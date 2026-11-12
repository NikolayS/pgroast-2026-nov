# Design pre-review: pg_xmin_horizon system view

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 7004](https://commitfest.postgresql.org/patch/7004/); [thread](https://www.postgresql.org/message-id/flat/AT-erSWn0vFw8TYLb84jqyfxgTV21gjwEo4YJDcses8qWtDkQK_pS6g2OIfVh4-L1PPn94w8-tE1mHZactVAgxPJ8vL96MXsyAnUg2Xz-Ks%3D%40scottray.io), 30 of 30 messages read; no code read.

## Verdict

**NEEDS-COMMITTER-EYES** (confidence: medium). No surviving objection questions the problem or the demand, and the three that survived are fixable level 2 points. The shape and name of the view (standards-compat-1) and the unguarded second copy of the horizon rules (maintenance-fit-1) need a decision from a committer who knows procarray.c. The v8 standby support (maintenance-fit-2) has no visible review. No committer has replied in 30 messages.

Not checked: The code was not read. Whether the v8 recovery path reads KnownAssignedXids correctly, and whether the classification matches ComputeXidHorizons() today, can only be judged from the patch.

## Prior attempts

None found before 2026-07-12.

## Objections a reviewer is likely to raise

### 1. Second copy of the horizon rules has no test that catches drift

A later patch changes which procs or slot xmins ComputeXidHorizons() counts, for example a new status flag or a new kind of horizon input, and does not touch procarrayfuncs.c. pg_xmin_horizon keeps showing the old classification. A DBA then terminates a backend the horizon already ignores and misses the real holder, and nobody notices until a field report arrives.

Message #1 (note 7) says the view loses fidelity if the two copies diverge. The only guards are comments, one shared macro (message #23, item 17) and a remaining keep-in-sync comment on slot xmins (message #24). Sami proposed a shared ComputeXidHorizonsData() collector so the SQL side would not reinterpret procarray rules (message #25), and message #27 declined it. Master having other keep-in-sync comments is a fair point, but those mostly guard code paths that tests cover, and the tests here only cover today's kinds.

**Question:** Which test or assertion fails when ComputeXidHorizons() and pg_get_xmin_horizon() classify the same proc or slot differently?

<sub>fixable, level 2, maintenance burden ("who maintains this in 10 years"); messages [#1](https://www.postgresql.org/message-id/AT-erSWn0vFw8TYLb84jqyfxgTV21gjwEo4YJDcses8qWtDkQK_pS6g2OIfVh4-L1PPn94w8-tE1mHZactVAgxPJ8vL96MXsyAnUg2Xz-Ks%3D%40scottray.io), [#20](https://www.postgresql.org/message-id/CAN12%2BYLWDpmYh4ndVa02Uyi1EPG%2Bd8s4PL-S8oBNYojYQxT35w%40mail.gmail.com), [#23](https://www.postgresql.org/message-id/qhODUcdKmqtOtwF4bW03SxPfKHda4EC7CEO2Ddj2jThrGRa8F1pDMA7LtG0R5JRo-2GK5RStXVP7tPP7WAUvfb_GCSN_I_YG7khDW859Fxg%3D%40scottray.io), [#24](https://www.postgresql.org/message-id/5x8VRERZxSIJ7thPBHMfEnE01ak_VVoMlCewFS2mhRB7q8Iy_3LP7qCtI_KVjd_bCMZ8_HDBmlabKuADPaic1KDJHAJMl3mSCaT4DgumPdE%3D%40scottray.io), [#25](https://www.postgresql.org/message-id/CAN12%2BYLOLcsePG8r9m9gTomXHbJ6ynd%2B-eLHmoN4eoBBQMSVKw%40mail.gmail.com), [#27](https://www.postgresql.org/message-id/i2q1k7cXS05oPhU_tBZfXKJ0tgqQ6unlVDrB8EhjjazAHmGhX3Qk9AAhybbf59b0yA_BH66GWzTc1hwvnnJb5WFApD1apw6DTrhAZSHqL3E%3D%40scottray.io); `maintenance-fit-1`</sub>

### 2. View name and row shape fixed before the per-horizon question is settled

Monitoring tools and runbooks adopt pg_xmin_horizon with per-source rows in v20. A later release adds the per-horizon summary that was also asked for. That summary cannot use the natural name, so users get two views whose names do not say which one holds horizons and which holds inputs.

Message #26 shows that two earlier committer messages ask for different shapes, and message #28 keeps per-source rows as the only interface. The singular name "pg_xmin_horizon" reads as the horizon values, but no row is a horizon: there are three horizons per database. The author rejected pg_vacuum_blockers in message #15 because names must describe contents, and the same rule applies to this name. No committer has picked a shape, and a system view name is hard to take back once tools depend on it.

**Question:** If a per-horizon view is added later, what would it be called, and why does pg_xmin_horizon describe per-source rows better than a name such as pg_xmin_horizon_sources?

<sub>fixable, level 2, SQL-standard conflict or non-standard syntax; messages [#14](https://www.postgresql.org/message-id/CALj2ACWiuAh0BmePj1J7U_KtK1KE8rY89kN5BvWD%2BqUjmzAsAg%40mail.gmail.com), [#15](https://www.postgresql.org/message-id/3TmV7r48RekS9lI-goS5ZDrVOfRrrNwFY1nnghf05L3Ays6_km2Qu0LWqCg_-LhUJrUbKjhpm9VI6pjB7aZTpIMEe_9kAkpL_j4Dj08KazY%3D%40scottray.io), [#20](https://www.postgresql.org/message-id/CAN12%2BYLWDpmYh4ndVa02Uyi1EPG%2Bd8s4PL-S8oBNYojYQxT35w%40mail.gmail.com), [#25](https://www.postgresql.org/message-id/CAN12%2BYLOLcsePG8r9m9gTomXHbJ6ynd%2B-eLHmoN4eoBBQMSVKw%40mail.gmail.com), [#26](https://www.postgresql.org/message-id/CAOVWO5pALh%3DMiP-RtEcy91K3t%2BOJKXkoTjCF1U9_uva28r3b_g%40mail.gmail.com), [#27](https://www.postgresql.org/message-id/i2q1k7cXS05oPhU_tBZfXKJ0tgqQ6unlVDrB8EhjjazAHmGhX3Qk9AAhybbf59b0yA_BH66GWzTc1hwvnnJb5WFApD1apw6DTrhAZSHqL3E%3D%40scottray.io), [#28](https://www.postgresql.org/message-id/dQhNr2KEwosovTvtTZYLEiW8XP-aTKRpBwtXyGO6hctiD9MPiV3e5RF3WiWndu-farqbW1NLJU-aL9lZ5qn1IKi-A9dKD-hjaY71xqC6UBs%3D%40scottray.io); `standards-compat-1`</sub>

### 3. Standby support enlarged an all-or-nothing patch that no committer has reviewed

The patch goes in with the v8 primary_xacts row that reads KnownAssignedXids under ProcArrayLock. A standby then reports a wrong or missing primary_xacts row after promotion or during a snapshot overflow. The committer who must fix it reviewed none of the recovery path, and the thread shows no review of it.

Message #1 planned standby support as a follow-up, and v8 folded it in (message #27), so the patch grew past the 1400 lines Bharath already found hard to review (message #14). The author declined to split it (message #15), and the brief notes the v8 standby code has no visible review. No committer has posted in 30 messages, although the patch reaches into procarray.c and backend_status.c, where errors are hard to see. A primary-only first commit, with standby support afterwards, matches the original plan.

**Question:** Can the primary-only view be committed first, with the KnownAssignedXids and primary_xacts support as a separately reviewed second patch?

<sub>fixable, level 2, maintenance burden ("who maintains this in 10 years"); messages [#1](https://www.postgresql.org/message-id/AT-erSWn0vFw8TYLb84jqyfxgTV21gjwEo4YJDcses8qWtDkQK_pS6g2OIfVh4-L1PPn94w8-tE1mHZactVAgxPJ8vL96MXsyAnUg2Xz-Ks%3D%40scottray.io), [#14](https://www.postgresql.org/message-id/CALj2ACWiuAh0BmePj1J7U_KtK1KE8rY89kN5BvWD%2BqUjmzAsAg%40mail.gmail.com), [#15](https://www.postgresql.org/message-id/3TmV7r48RekS9lI-goS5ZDrVOfRrrNwFY1nnghf05L3Ays6_km2Qu0LWqCg_-LhUJrUbKjhpm9VI6pjB7aZTpIMEe_9kAkpL_j4Dj08KazY%3D%40scottray.io), [#27](https://www.postgresql.org/message-id/i2q1k7cXS05oPhU_tBZfXKJ0tgqQ6unlVDrB8EhjjazAHmGhX3Qk9AAhybbf59b0yA_BH66GWzTc1hwvnnJb5WFApD1apw6DTrhAZSHqL3E%3D%40scottray.io); `maintenance-fit-2`</sub>

## Questions for the author

1. Which test or assertion fails when ComputeXidHorizons() and pg_get_xmin_horizon() classify the same proc or slot differently?
2. If a per-horizon view is added later, what would it be called, and why does pg_xmin_horizon fit per-source rows better than a name such as pg_xmin_horizon_sources?
3. Can the primary-only view be committed first, with the KnownAssignedXids and primary_xacts support as a separately reviewed second patch?
4. Who has reviewed the v8 recovery path, and which tests cover the primary_xacts row?
5. Why was the shared collector proposed in message #25 declined, given that it would remove the second copy of the classification rules?

## Touch points

- **WAL / replication / standby:** The SRF reads replication slot effective xmins and, during recovery, KnownAssignedXids for the primary_xacts row.
- **catalog changes:** The patch adds the pg_xmin_horizon system view and a pg_proc entry for pg_get_xmin_horizon().
- **ABI / extension API:** The patch changes proc.h, procarray.h and backend_status.h, which extensions include.
- **SQL-standard conflict or non-standard syntax:** see objection standards-compat-1: View name and row shape fixed before the per-horizon question is settled
- **security / privileges:** The view and function are revoked from PUBLIC and granted to pg_read_all_stats, with no per-row check.
- **maintenance burden ("who maintains this in 10 years"):** procarrayfuncs.c repeats the classification rules of ComputeXidHorizons(), guarded by comments and one shared macro.
- **performance on the hot path:** The patch touches procarray.c near ComputeXidHorizons(), and each call takes ProcArrayLock in shared mode.

## Who would commit it, and when

- Owner: silent. It needs a committer who knows procarray.c, snapshot horizons and replication slot xmin handling. None has posted in the visible thread.
- Priority: needs champion; audience a sizeable minority; review cost medium. Demand is shown by a field report, commercial tools and earlier committer requests, and three reviewers have gone through the patch. It is waiting for a committer to choose the row shape and to accept a 1400-line patch that touches procarray.c.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 3 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
