# Design pre-review: Fix slot synchronization with two_phase decoding enabled

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 5664](https://commitfest.postgresql.org/patch/5664/); [thread](https://www.postgresql.org/message-id/flat/TYAPR01MB5724CC7C288535BBCEEE65DA94A72%40TYAPR01MB5724.jpnprd01.prod.outlook.com), 100 of 100 messages read; no code read.

## Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium). The HEAD fix is committed and the bug on 17 is real, so the goal is not in question. The decisive objection is prior-attempts-1: v19 makes a minor release reject `create subscription` statements that dumps from older 17.x contain. Messages #94 and #99 leave this open, and message #100 proposes a different design. The list must choose a design for 17 before code review of v19 is useful.

Not checked: The v19 patches for 17 were not read, so the exact checks, the binary-upgrade exception and the pg_dump change are judged from the thread only. The message #100 idea has no patch to judge.

## Prior attempts

- [Revert addition of PGDLLEXPORT in PG_FUNCTION_INFO_V1 macro.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=4f52fd3c6d28a4380a5afc51ae6f774c91837a38) (related; Reverted, 2016-10-12). "Third-party authors would not thank us for breaking their code in a minor release." Different now: P23 reverted a macro change that broke extension builds in back branches. This proposal breaks restore of older 17.x dumps through new option checks, so P23 is only a loose analogy.

## Objections a reviewer is likely to raise

### 1. Minor release makes dumps from older 17.x releases fail to restore

A user on 17.5 runs pg_dump for backup and the dump contains create subscription ... with (two_phase = true, failover = true). After a routine upgrade to the fixed 17.x, the restore stops with an error at that statement. The user finds this only when they need the backup.

[P23] records a revert because a change would break third-party code in a minor release. The same reason applies to a minor release that rejects SQL that older 17.x dumps contain. Amit raised the case in message #81 and Sawada in messages #83, #94 and #99. Messages #89 and #96 answer with the docs on binary replacement and on using the newer pg_dump. Message #99 notes that pg_dump is also used for plain backups, and message #100 proposes a different design, so the thread has not settled the point.

Precedent: [Revert addition of PGDLLEXPORT in PG_FUNCTION_INFO_V1 macro.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=4f52fd3c6d28a4380a5afc51ae6f774c91837a38) (Reverted, 2016-10-12)

**Question:** How does a dump taken by 17.5 pg_dump restore on the fixed 17.x, or why is breaking that restore acceptable in a minor release?

<sub>fixable, level 1b, backpatchability; messages [#81](https://www.postgresql.org/message-id/CAA4eK1KiGU5Zr_S2g8w-8dUm-PRiZc72BgMnLdS83u7sgnctcg%40mail.gmail.com), [#83](https://www.postgresql.org/message-id/CAD21AoCvCw42m6UxdFA%3D3FgUaz_Eb%2BLvaYFNTbagMQJc1tHphQ%40mail.gmail.com), [#89](https://www.postgresql.org/message-id/CABdArM4pp6HdiENq%2BE1WrcD%3DckH62kT2c6S%2BV%3DW1Qoxhtf4_jg%40mail.gmail.com), [#94](https://www.postgresql.org/message-id/CAD21AoAOkfLCiSamfBYVtFiXXLr3LVCdpBjvm6ssYA2O_ZVF%3Dw%40mail.gmail.com), [#96](https://www.postgresql.org/message-id/CAJpy0uA88qiY1EXmtppchFNxGEqxaB1X_z6YLEmpM_f5j1i9UQ%40mail.gmail.com), [#99](https://www.postgresql.org/message-id/CAD21AoCnurDW-L-qoiVD3XDQW6iPGSa3WgW5mz4zU-7dDz5Dpg%40mail.gmail.com), [#100](https://www.postgresql.org/message-id/CAJpy0uDuGz0B%2BzACCiWt9GOGA2jH_%2Br_zM%3D9c2h2MKacd%2BqYkQ%40mail.gmail.com); `prior-attempts-1`</sub>

## Questions for the author

1. How does a dump taken by 17.5 pg_dump with failover = true and two_phase = true restore on the fixed 17.x without manual editing?
2. Which design does the CommitFest entry now track for 17: the v19 restriction, or the idea in message #100 of reporting failover as false?
3. After the minor update, how does a 17 user find which existing failover slots with two_phase are still unsafe, given that 17 does not show two_phase_at?
4. Under the message #100 idea, what happens to a slot the standby has already synced when the primary starts to report failover as false?
5. How does a user who restores a dump from the new pg_dump learn that failover was turned off for the subscription?

## Touch points

- **on-disk format / pg_upgrade:** Binary upgrade mode keeps creating slots with both options, and existing slot state on disk passes through the minor update unchanged.
- **WAL / replication / standby:** The fix changes slot sync to the standby and which logical slots may have failover enabled.
- **catalog changes:** The HEAD fix adds the two_phase_at column to pg_replication_slots, which is why it cannot go to 17.
- **SQL-standard conflict or non-standard syntax:** CREATE SUBSCRIPTION and pg_create_logical_replication_slot() start to reject an option combination that 17.0 to 17.5 accept.
- **pg_dump / restore round-trip:** pg_dump writes failover = false for subscriptions with both options, and older 17.x dumps fail to restore.
- **backpatchability:** The entry targets 17 only and changes the behaviour of existing commands in a minor release.
- **maintenance burden ("who maintains this in 10 years"):** The v19 checks exist only on the 17 branch, because HEAD fixes the bug by syncing two_phase_at.

## Who would commit it, and when

- Owner: qualified committer engaged. Amit Kapila committed the HEAD fix and reviews the 17 patch. The 17 change needs a committer who knows slot sync and who accepts the minor-release behaviour change.
- Priority: needs design rethink; audience niche; review cost medium. Only 17 users who set both failover and two_phase are affected, and the trigger on 17 is rare. v19 has review, but the dump restore objection is open and message #100 proposes another design, so the 17 design must be agreed first.

---

[roast-full.md](roast-full.md) has 3 more objection(s) that held, 6 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
