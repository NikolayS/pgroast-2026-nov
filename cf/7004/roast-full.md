# Design pre-review: pg_xmin_horizon system view

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 7004](https://commitfest.postgresql.org/patch/7004/) (Needs review); authors: Scott Ray (scottray)
- Thread: [pg_xmin_horizon: a system view of everything pinning the xmin horizon](https://www.postgresql.org/message-id/flat/AT-erSWn0vFw8TYLb84jqyfxgTV21gjwEo4YJDcses8qWtDkQK_pS6g2OIfVh4-L1PPn94w8-tE1mHZactVAgxPJ8vL96MXsyAnUg2Xz-Ks%3D%40scottray.io), started 2026-07-12
- Messages read: 30 of 30
- Patch prose read: v10-0001-Add-pg_xmin_horizon-view-showing-per-input-horiz.patch, v1-0001-Add-pg_xmin_horizon-view-showing-per-input-horizo.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-COMMITTER-EYES** (confidence: medium)

No surviving objection questions the problem or the demand, and the three that survived are fixable level 2 points. The shape and name of the view (standards-compat-1) and the unguarded second copy of the horizon rules (maintenance-fit-1) need a decision from a committer who knows procarray.c. The v8 standby support (maintenance-fit-2) has no visible review. No committer has replied in 30 messages.

No level of the review pyramid fails.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: passed
- Levels 3-5 (implementation): out of scope

Provisional: The code was not read. Whether the v8 recovery path reads KnownAssignedXids correctly, and whether the classification matches ComputeXidHorizons() today, can only be judged from the patch.

## The proposal, restated

**Problem.** Finding what holds back the xmin horizon means joining pg_stat_activity, pg_replication_slots, pg_prepared_xacts and pg_stat_replication, and even that join is only approximate. Slot effective xmins and the proc flags that make ComputeXidHorizons() skip a backend are not visible from SQL. DBAs who do not know all the sources may miss the holder, or terminate a plain VACUUM that pins nothing.

**Proposed design.** A new SRF, pg_get_xmin_horizon(), and a view on top of it return one row per horizon input with its contribution to the shared, catalog and data horizons. The SRF reads procs in one ProcArrayLock pass and slots in a separate pass, so rows are not guaranteed consistent with each other. The view does no ranking, and it includes inputs that pin nothing.

**User-visible surface.**
- New system view pg_xmin_horizon with columns kind, pid, slot_name, gid, datid, datname, xid, shared_xmin, catalog_xmin, data_xmin, xact_start.
- New function pg_get_xmin_horizon(), restricted to pg_read_all_stats like the view.
- kind values: backend, replication_slot, prepared_xact, standby_feedback, and primary_xacts during recovery.
- maintenance.sgml wraparound steps now point to pg_xmin_horizon and no longer to pg_stat_activity, pg_prepared_xacts and pg_replication_slots.
- maintenance.sgml now says datfrozenxid is no newer than, and normally equal to, the minimum relfrozenxid.

**What the author claims.**
- A UNION over existing views can only approximate the answer, because slot effective xmins and horizon classification are not exposed.
- The existing wraparound doc steps are wrong, since pg_stat_activity shows backend_xmin for backends the horizons skip.
- Commercial tools and Andres's wishlist show demand for this view.
- A per-horizon summary can be built from per-source rows, but per-source rows cannot be rebuilt from a summary.
- Keeping the logic in C, next to ComputeXidHorizons(), makes it easier to review and keep in sync.

**Thread state.** Solai, Surya and Bharath reviewed several versions, and Bharath and Surya accepted the C-function approach. Sami proposed a per-horizon base function, Surya suggested providing both shapes, and Scott declined; v10 is posted and no committer has replied.

## 2. Prior attempts

No precedent found (corpus searched for items before 2026-07-12; 24 candidates retrieved, none is the same or a similar idea).

## 3. Objections a reviewer is likely to raise

### 1. Second copy of the horizon rules has no test that catches drift

fixable · level 2, rubric item 9 · maintenance burden ("who maintains this in 10 years") · `maintenance-fit-1`

A later patch changes which procs or slot xmins ComputeXidHorizons() counts, for example a new status flag or a new kind of horizon input, and does not touch procarrayfuncs.c. pg_xmin_horizon keeps showing the old classification. A DBA then terminates a backend the horizon already ignores and misses the real holder, and nobody notices until a field report arrives.

Message #1 (note 7) says the view loses fidelity if the two copies diverge. The only guards are comments, one shared macro (message #23, item 17) and a remaining keep-in-sync comment on slot xmins (message #24). Sami proposed a shared ComputeXidHorizonsData() collector so the SQL side would not reinterpret procarray rules (message #25), and message #27 declined it. Master having other keep-in-sync comments is a fair point, but those mostly guard code paths that tests cover, and the tests here only cover today's kinds.

Thread: [#1 (Scott Ray, 2026-07-12)](https://www.postgresql.org/message-id/AT-erSWn0vFw8TYLb84jqyfxgTV21gjwEo4YJDcses8qWtDkQK_pS6g2OIfVh4-L1PPn94w8-tE1mHZactVAgxPJ8vL96MXsyAnUg2Xz-Ks%3D%40scottray.io), [#20 (Sami Imseih, 2026-09-18)](https://www.postgresql.org/message-id/CAN12%2BYLWDpmYh4ndVa02Uyi1EPG%2Bd8s4PL-S8oBNYojYQxT35w%40mail.gmail.com), [#23 (Scott Ray, 2026-09-27)](https://www.postgresql.org/message-id/qhODUcdKmqtOtwF4bW03SxPfKHda4EC7CEO2Ddj2jThrGRa8F1pDMA7LtG0R5JRo-2GK5RStXVP7tPP7WAUvfb_GCSN_I_YG7khDW859Fxg%3D%40scottray.io), [#24 (Scott Ray, 2026-09-27)](https://www.postgresql.org/message-id/5x8VRERZxSIJ7thPBHMfEnE01ak_VVoMlCewFS2mhRB7q8Iy_3LP7qCtI_KVjd_bCMZ8_HDBmlabKuADPaic1KDJHAJMl3mSCaT4DgumPdE%3D%40scottray.io), [#25 (Sami Imseih, 2026-09-28)](https://www.postgresql.org/message-id/CAN12%2BYLOLcsePG8r9m9gTomXHbJ6ynd%2B-eLHmoN4eoBBQMSVKw%40mail.gmail.com), [#27 (Scott Ray, 2026-10-05)](https://www.postgresql.org/message-id/i2q1k7cXS05oPhU_tBZfXKJ0tgqQ6unlVDrB8EhjjazAHmGhX3Qk9AAhybbf59b0yA_BH66GWzTc1hwvnnJb5WFApD1apw6DTrhAZSHqL3E%3D%40scottray.io)

**Question:** Which test or assertion fails when ComputeXidHorizons() and pg_get_xmin_horizon() classify the same proc or slot differently?

Counter-check: held. Message #1 note 7 states the drift risk, and the guards shown are one macro and comments. The material names no test or assertion that fails when the two classifications diverge.

### 2. View name and row shape fixed before the per-horizon question is settled

fixable · level 2, rubric item 10 · SQL-standard conflict or non-standard syntax, maintenance burden ("who maintains this in 10 years") · `standards-compat-1`

Monitoring tools and runbooks adopt pg_xmin_horizon with per-source rows in v20. A later release adds the per-horizon summary that was also asked for. That summary cannot use the natural name, so users get two views whose names do not say which one holds horizons and which holds inputs.

Message #26 shows that two earlier committer messages ask for different shapes, and message #28 keeps per-source rows as the only interface. The singular name "pg_xmin_horizon" reads as the horizon values, but no row is a horizon: there are three horizons per database. The author rejected pg_vacuum_blockers in message #15 because names must describe contents, and the same rule applies to this name. No committer has picked a shape, and a system view name is hard to take back once tools depend on it.

Thread: [#14 (Bharath Rupireddy, 2026-09-01)](https://www.postgresql.org/message-id/CALj2ACWiuAh0BmePj1J7U_KtK1KE8rY89kN5BvWD%2BqUjmzAsAg%40mail.gmail.com), [#15 (Scott Ray, 2026-09-05)](https://www.postgresql.org/message-id/3TmV7r48RekS9lI-goS5ZDrVOfRrrNwFY1nnghf05L3Ays6_km2Qu0LWqCg_-LhUJrUbKjhpm9VI6pjB7aZTpIMEe_9kAkpL_j4Dj08KazY%3D%40scottray.io), [#20 (Sami Imseih, 2026-09-18)](https://www.postgresql.org/message-id/CAN12%2BYLWDpmYh4ndVa02Uyi1EPG%2Bd8s4PL-S8oBNYojYQxT35w%40mail.gmail.com), [#25 (Sami Imseih, 2026-09-28)](https://www.postgresql.org/message-id/CAN12%2BYLOLcsePG8r9m9gTomXHbJ6ynd%2B-eLHmoN4eoBBQMSVKw%40mail.gmail.com), [#26 (surya poondla, 2026-09-29)](https://www.postgresql.org/message-id/CAOVWO5pALh%3DMiP-RtEcy91K3t%2BOJKXkoTjCF1U9_uva28r3b_g%40mail.gmail.com), [#27 (Scott Ray, 2026-10-05)](https://www.postgresql.org/message-id/i2q1k7cXS05oPhU_tBZfXKJ0tgqQ6unlVDrB8EhjjazAHmGhX3Qk9AAhybbf59b0yA_BH66GWzTc1hwvnnJb5WFApD1apw6DTrhAZSHqL3E%3D%40scottray.io), [#28 (Scott Ray, 2026-10-05)](https://www.postgresql.org/message-id/dQhNr2KEwosovTvtTZYLEiW8XP-aTKRpBwtXyGO6hctiD9MPiV3e5RF3WiWndu-farqbW1NLJU-aL9lZ5qn1IKi-A9dKD-hjaY71xqC6UBs%3D%40scottray.io)

**Question:** If a per-horizon view is added later, what would it be called, and why does pg_xmin_horizon describe per-source rows better than a name such as pg_xmin_horizon_sources?

Counter-check: held. Messages #25 and #26 show two shapes were asked for, and no committer has chosen. The author's own rule in message #15, that names denote contents, leaves open why a singular horizon name fits per-source rows.

### 3. Standby support enlarged an all-or-nothing patch that no committer has reviewed

fixable · level 2, rubric item 16 · maintenance burden ("who maintains this in 10 years"), WAL / replication / standby · `maintenance-fit-2`

The patch goes in with the v8 primary_xacts row that reads KnownAssignedXids under ProcArrayLock. A standby then reports a wrong or missing primary_xacts row after promotion or during a snapshot overflow. The committer who must fix it reviewed none of the recovery path, and the thread shows no review of it.

Message #1 planned standby support as a follow-up, and v8 folded it in (message #27), so the patch grew past the 1400 lines Bharath already found hard to review (message #14). The author declined to split it (message #15), and the brief notes the v8 standby code has no visible review. No committer has posted in 30 messages, although the patch reaches into procarray.c and backend_status.c, where errors are hard to see. A primary-only first commit, with standby support afterwards, matches the original plan.

Thread: [#1 (Scott Ray, 2026-07-12)](https://www.postgresql.org/message-id/AT-erSWn0vFw8TYLb84jqyfxgTV21gjwEo4YJDcses8qWtDkQK_pS6g2OIfVh4-L1PPn94w8-tE1mHZactVAgxPJ8vL96MXsyAnUg2Xz-Ks%3D%40scottray.io), [#14 (Bharath Rupireddy, 2026-09-01)](https://www.postgresql.org/message-id/CALj2ACWiuAh0BmePj1J7U_KtK1KE8rY89kN5BvWD%2BqUjmzAsAg%40mail.gmail.com), [#15 (Scott Ray, 2026-09-05)](https://www.postgresql.org/message-id/3TmV7r48RekS9lI-goS5ZDrVOfRrrNwFY1nnghf05L3Ays6_km2Qu0LWqCg_-LhUJrUbKjhpm9VI6pjB7aZTpIMEe_9kAkpL_j4Dj08KazY%3D%40scottray.io), [#27 (Scott Ray, 2026-10-05)](https://www.postgresql.org/message-id/i2q1k7cXS05oPhU_tBZfXKJ0tgqQ6unlVDrB8EhjjazAHmGhX3Qk9AAhybbf59b0yA_BH66GWzTc1hwvnnJb5WFApD1apw6DTrhAZSHqL3E%3D%40scottray.io)

**Question:** Can the primary-only view be committed first, with the KnownAssignedXids and primary_xacts support as a separately reviewed second patch?

Counter-check: held. The reply in message #15 answers a per-kind split, but a primary-only first patch is separately committable, as v1 to v7 were. The promotion and overflow scenario is speculative, so the objection rests on patch shape and the unreviewed v8 standby code.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | no |  |
| WAL / replication / standby | **yes** | The SRF reads replication slot effective xmins and, during recovery, KnownAssignedXids for the primary_xacts row. |
| wire protocol | no |  |
| catalog changes | **yes** | The patch adds the pg_xmin_horizon system view and a pg_proc entry for pg_get_xmin_horizon(). |
| ABI / extension API | **yes** | The patch changes proc.h, procarray.h and backend_status.h, which extensions include. |
| SQL-standard conflict or non-standard syntax | **yes** | see objection standards-compat-1: View name and row shape fixed before the per-horizon question is settled |
| new GUC | no |  |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | no |  |
| security / privileges | **yes** | The view and function are revoked from PUBLIC and granted to pg_read_all_stats, with no per-row check. |
| backpatchability | no |  |
| maintenance burden ("who maintains this in 10 years") | **yes** | procarrayfuncs.c repeats the classification rules of ComputeXidHorizons(), guarded by comments and one shared macro. |
| performance on the hot path | **yes** | The patch touches procarray.c near ComputeXidHorizons(), and each call takes ProcArrayLock in shared mode. |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. Which test or assertion fails when ComputeXidHorizons() and pg_get_xmin_horizon() classify the same proc or slot differently?
2. If a per-horizon view is added later, what would it be called, and why does pg_xmin_horizon fit per-source rows better than a name such as pg_xmin_horizon_sources?
3. Can the primary-only view be committed first, with the KnownAssignedXids and primary_xacts support as a separately reviewed second patch?
4. Who has reviewed the v8 recovery path, and which tests cover the primary_xacts row?
5. Why was the shared collector proposed in message #25 declined, given that it would remove the second copy of the classification rules?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** silent. It needs a committer who knows procarray.c, snapshot horizons and replication slot xmin handling. None has posted in the visible thread.
- **Liability:** The owner signs for a permanent system view whose rows must match ComputeXidHorizons() in every later release, on primaries and on standbys, and for wraparound documentation that now sends users to it.
- **Author follow-up:** The author posted ten versions in four months, answered each reviewer and kept the patch rebased. That is a good sign for post-commit fixes.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** needs champion
- **Audience:** a sizeable minority
- **Maintenance burden:** medium
- **Review cost:** medium
- **Why:** Demand is shown by a field report, commercial tools and earlier committer requests, and three reviewers have gone through the patch. It is waiting for a committer to choose the row shape and to accept a 1400-line patch that touches procarray.c.

## Appendix

### Refuted in the counter-check

- `standards-compat-2` All-or-nothing access differs from the per-row model of session views. Refuted (no failure scenario): The role in the scenario still sees its own backend_xmin in pg_stat_activity, so it loses nothing. A restrictive default can be loosened later with a grant, and Bharath called the asymmetry "probably okay" in message #16.
- `standards-compat-3` datid means something different during recovery. Refuted (already handled): datid names the same database on primary and standby. The scoping differs because ComputeXidHorizons() counts backends of all databases during recovery, and the v10 docs say so. The missing review of standby support is covered by maintenance-fit-2.
- `maintenance-fit-3` View columns freeze the internal shared, catalog and data split. Refuted (no failure scenario): The scenario depends on an unnamed future regrouping of horizons. pg_replication_slots already publishes the xmin and catalog_xmin split, and the per-horizon shape in message #25 exposes the same three classes, so the cost is not specific to this design.

### Review angles

- prior-attempts: 0 objection(s). No precedent found, and no earlier thread in the material records a rejection reason that applies here. Message #26 reports two earlier committer sketches, one per-source and one per-horizon; the proposal follows the per-source one, and the open shape question belongs to the design angle.
- standards-compat: 3 objection(s). Objections listed; no precedent was retrieved, so none is cited.
- replication-ondisk: 0 objection(s). The proposal adds a read-only view and SRF over shared memory, with no WAL records, no on-disk format change and no new persistent state; the only catalog change is a new pg_proc entry and view, which pg_upgrade and pg_dump handle normally. No reviewer has yet looked at the v8 recovery path that reads KnownAssignedXids and emits the primary_xacts row, and only the code can show whether it is correct.
- security: 0 objection(s). No security objection. The view and pg_get_xmin_horizon() are revoked from PUBLIC and need pg_read_all_stats, which already sees the same pids, xids, xmins, xact_start values, slot xmins and gids through pg_stat_activity, pg_replication_slots and pg_prepared_xacts. The SRF takes ProcArrayLock in shared mode once per call, and only privileged roles can call it. The missing per-row HAS_PGSTAT_PERMISSIONS check (message #17) matters only if a DBA grants EXECUTE directly, and that exposes only xact_start, following the pg_shmem_allocations convention (message #22).
- maintenance-fit: 3 objection(s). The precedent search found no related proposal. Hot-path cost looks negligible, since the patch only shares a macro with ComputeXidHorizons(), but that cannot be confirmed without the code.
- already-exists: 0 objection(s). Existing views cannot do this job: pg_stat_activity shows backend_xmin for backends the horizons skip, slot effective xmins are not exposed, and the standby primary_xacts input has no SQL surface. Message #10 proposed adding columns to existing views instead, and the reviewer accepted the author's answer in message #14, so no core feature, contrib module or extension covers the need.
- demand: 0 objection(s). The material shows real, independent demand: a field report (message #1), commercial tools from several vendors (messages #1, #27), and committer requests for such a view in 2020 and 2023 (message #26). The open question is the view's shape, which is a design matter for other angles.

### Precedents retrieved

- P1: [Revert changes to CONCURRENTLY that "sped up" Xmin advance](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=e28bb885196916b0a3d898ae4f2be0e38108d81b) (revert, Reverted, 2022-05-31) - judged not relevant
- P2: [Revert "Test that vacuum removes tuples older than OldestXmin"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=efcbb76efe406d59c2ba8b4a09e04c01158ba575) (revert, Reverted, 2024-07-22) - judged not relevant
- P3: [Track Oldest Initialized WAL Buffer Page](https://commitfest.postgresql.org/patch/4210/) (cf, Returned with feedback, 2023-08-03) - judged not relevant; reason summarised from its thread
- P4: [Lockable views](https://commitfest.postgresql.org/patch/1334/) (cf, Returned with feedback, 2017-11-29) - judged not relevant; reason summarised from its thread
- P5: [Triggers on Materialized Views](https://commitfest.postgresql.org/patch/1948/) (cf, Returned with feedback, 2019-04-01) - judged not relevant
- P6: [Allow VIEW/RULE recompilation when the underlying tables change](https://wiki.postgresql.org/wiki/Not_Worth_Doing#Views_and_Rules) (do_not_want, Not worth doing, date unknown) - judged not relevant; reason summarised from its thread
- P7: [Triggers on materialized views](https://commitfest.postgresql.org/patch/1953/) (cf, Withdrawn, 2019-01-04) - judged not relevant
- P8: [Changing the autovacuum launcher scheduling; oldest table first algorithm](https://commitfest.postgresql.org/patch/1573/) (cf, Returned with feedback, 2018-11-30) - judged not relevant; reason summarised from its thread
- P9: [Incremental Materialized View Maintenance](https://commitfest.postgresql.org/patch/2138/) (cf, Returned with feedback, 2022-10-12) - judged not relevant; reason summarised from its thread
- P10: [Temporary materialized views](https://commitfest.postgresql.org/patch/1951/) (cf, Returned with feedback, 2019-03-15) - judged not relevant
- P11: [This adds a new GUC parameter that automatically rolls back prepared transactions that have remained unresolved beyond the configured timeout. The cleanup is performed by the checkpointer process.](https://commitfest.postgresql.org/patch/6611/) (cf, Rejected, 2026-07-01) - judged not relevant; reason summarised from its thread
- P12: [Track invalidation_reason in pg_replication_slots.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=6ae701b4378db2284c77314560e95a93d0ba9484) (revert, Partially reverted, 2024-03-22) - judged not relevant
- P13: [Add tracking of backend memory allocated to pg_stat_activity](https://commitfest.postgresql.org/patch/3865/) (cf, Withdrawn, 2023-01-05) - judged not relevant; reason summarised from its thread
- P14: [Revert updatable views](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=5fe3da927b2857761f9206d9fc597cf5321d9638) (revert, Reverted, 2009-01-27) - judged not relevant
- P15: [Add LWLock blocker(s) information](https://commitfest.postgresql.org/patch/2576/) (cf, Returned with feedback, 2020-11-18) - judged not relevant; reason summarised from its thread
- P16: [SQL/JSON: functions](https://commitfest.postgresql.org/patch/1472/) (cf, Returned with feedback, 2020-09-17) - judged not relevant
- P17: [Reuse Workers and Replication Slots during Logical Replication](https://commitfest.postgresql.org/patch/3784/) (cf, Returned with feedback, 2024-01-10) - judged not relevant
- P18: [pg_stat_activity: avoid showing state=active with wait_event=ClientRead](https://commitfest.postgresql.org/patch/3760/) (cf, Returned with feedback, 2022-11-30) - judged not relevant
- P19: [System username in pg_stat_activity](https://commitfest.postgresql.org/patch/4769/) (cf, Returned with feedback, 2024-10-07) - judged not relevant
- P20: [Error "initial slot snapshot too large" in create replication slot](https://commitfest.postgresql.org/patch/3445/) (cf, Withdrawn, 2024-01-16) - judged not relevant
- P21: [Patch to warn about any potential orphaned prepared transactions](https://commitfest.postgresql.org/patch/2505/) (cf, Withdrawn, 2020-04-17) - judged not relevant
- P22: [Improve monitoring of shared memory allocations](https://commitfest.postgresql.org/patch/5620/) (cf, Rejected, 2026-04-07) - judged not relevant
- P23: [Add function to release an allocated SQLDA](https://commitfest.postgresql.org/patch/1669/) (cf, Returned with feedback, 2018-11-30) - judged not relevant
- P24: [SQL/JSON json_table() function PLAN clause](https://commitfest.postgresql.org/patch/6590/) (cf, Withdrawn, 2026-05-19) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 14; wall time: 142 s; cost reported by the CLI: $3.98
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 6 from cache, 2 summarised now, 0 failed; quotes: 1 verified, 0 dropped; relevance pass: done, 0 of 24 kept
- Tone check: 78 text field(s) checked, 1 finding(s), 1 sentence(s) rewritten (repair call: done), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T03:13:15.727Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
