# Design pre-review: Timing issue in pg_basebackup

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 7613](https://commitfest.postgresql.org/patch/7613/) (Needs review); authors: Jesper St John (stjohn)
- Thread: [Timing issue in pg_basebackup](https://www.postgresql.org/message-id/flat/ddfa03dcf26f366ae493e8cc41bd479a%40stjohn.se), started 2026-10-19
- Messages read: 6 of 6
- Patch prose read: v3-0001-pg_basebackup-Create-the-WAL-streaming-slot-befor.patch, v1-0001-pg_basebackup-Create-the-WAL-streaming-slot-befor.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-COMMITTER-EYES** (confidence: medium)

The bug is reproduced on v16 to v18 and a reviewer confirmed it in message #2, so the goal is not in question. One objection survived, maintenance-fit-1: the thread does not compare this client-side reordering with a server-side WAL reservation in BASE_BACKUP. That is a fixable design point and does not block the idea. What remains is back-patch scope and the exit-time slot drop, which need a committer who knows pg_basebackup.

No level of the review pyramid fails.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: passed
- Levels 3-5 (implementation): out of scope

Provisional: Without the code, nobody can judge whether the check in drop_lingering_replication_slot_atexit() is right, or how the handler treats --slot and --create-slot. Only 3 of 6 thread messages were shown.

## The proposal, restated

**Problem.** With --wal-method=stream, pg_basebackup creates its replication slot only after BASE_BACKUP returns the start position, so a checkpoint in between can remove the needed WAL. The backup then fails with "requested WAL segment ... has already been removed". The author hit this running two pg_basebackup jobs at once from ansible, on v16, v17 and v18.

**Proposed design.** pg_basebackup opens the WAL streaming connection and creates its slot before it sends BASE_BACKUP, so the slot reserves the WAL from the start. An atexit handler drops the new slot if pg_basebackup exits before the background WAL receiver takes over the connection. New TAP tests use an injection point to hold slot creation, and a bogus --manifest-checksums value to fail BASE_BACKUP after the slot exists.

**User-visible surface.**
- Changed order: the slot used by pg_basebackup --wal-method=stream now exists before BASE_BACKUP starts, so it is visible for longer.
- pg_basebackup now drops its new slot on early exit.

**What the author claims.**
- The race also happens when any session runs a checkpoint, not only a second backup.
- Creating the slot first makes it reserve all the WAL the backup needs.
- Lingering slots can already happen today; the patch only raises the risk, which the atexit handler addresses.
- With --checkpoint=spread the slot stays open longer, which the author lists as a possible caveat.
- The author used an LLM to help write the Perl test.

**Thread state.** Kirill Reshke agreed the bug is real and asked for slot cleanup and a clearer comment; the author then sent v2 in two patches and v3 merged into one. Nobody has replied to v2 or v3, and the author is unsure whether the check in drop_lingering_replication_slot_atexit() is right.

## 2. Prior attempts

No precedent found (corpus searched for items before 2026-10-19; 17 candidates retrieved, none is the same or a similar idea).

## 3. Objections a reviewer is likely to raise

### 1. Client-side reordering leaves the race for other BASE_BACKUP clients

fixable · level 2, rubric item 11 · design, maintenance burden ("who maintains this in 10 years"), backpatchability · `maintenance-fit-1`

An operator runs an older pg_basebackup binary, or a third-party tool that speaks BASE_BACKUP and streams WAL separately, against a patched server. A concurrent checkpoint removes WAL after the start position, and the backup still fails with "requested WAL segment ... has already been removed". The operator finds out only when the backup fails.

The race exists because the server reports a start position without reserving WAL past it, yet the fix lives in one client. Every other client of the replication protocol has to copy the same reordering and the same cleanup logic. The proposal does not say whether a server-side guarantee was considered, such as BASE_BACKUP holding WAL from its start position until the backup ends. Nothing in the thread (messages #1 to #6) compares the two approaches.

Thread: [#1 (Jesper St John, 2026-10-19)](https://www.postgresql.org/message-id/ddfa03dcf26f366ae493e8cc41bd479a%40stjohn.se), [#5 (Jesper St John, 2026-10-29)](https://www.postgresql.org/message-id/b324489a123ae3b43e2eca9a5833157e%40stjohn.se), [#6 (Jesper St John, 2026-11-09)](https://www.postgresql.org/message-id/80143a07e6732576c9f815a3058fa31d%40stjohn.se)

**Question:** Why is a client-side reordering better than having BASE_BACKUP itself keep WAL from the start position, which would also protect older clients and other tools?

Counter-check: held. The patch does not make other clients worse, so the scenario describes today's behaviour. But the thread never compares a server-side WAL reservation with the client-side reordering, and a committer would likely ask for that comparison.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | no |  |
| WAL / replication / standby | **yes** | The replication slot now exists before BASE_BACKUP starts, so it holds WAL for longer, mainly with --checkpoint=spread. |
| wire protocol | no |  |
| catalog changes | no |  |
| ABI / extension API | no |  |
| SQL-standard conflict or non-standard syntax | no |  |
| new GUC | no |  |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | no |  |
| security / privileges | no |  |
| backpatchability | **yes** | The bug is reported on v16, v17 and v18, but the proposal names no target branches. |
| maintenance burden ("who maintains this in 10 years") | **yes** | The patch adds an atexit handler in pg_basebackup.c that drops the slot, plus an injection-point TAP test. |
| performance on the hot path | no |  |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. Why is reordering in pg_basebackup better than having BASE_BACKUP keep WAL from its start position, which would also protect older clients and other tools?
2. Which branches is this fix meant for: master only, or also v16 through v18?
3. How does the new order behave with --slot, with --create-slot, and with the default temporary slot, and which of these does the atexit handler drop?
4. What does the atexit handler do when the connection is already lost, or when the slot drop itself fails?
5. Does the new order work against every older server version that pg_basebackup still supports?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** silent. It needs a committer who knows pg_basebackup and replication slot handling; none has spoken in the visible thread.
- **Liability:** The committer signs for changed slot timing in pg_basebackup, possibly on three stable branches, and for an exit-time slot drop whose check the author is unsure about.
- **Author follow-up:** The author answered review within a day, sent v2 and v3, and registered the CommitFest entry.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** proceed
- **Audience:** a sizeable minority
- **Maintenance burden:** low
- **Review cost:** low
- **Why:** The race affects pg_basebackup --wal-method=stream when a checkpoint runs at the wrong moment, and it is reproduced on three releases. The patch is about 13 KiB in one client program plus tests, with no new user-visible option.

## Appendix

### Refuted in the counter-check

- `replication-ondisk-1` Permanent slot from --create-slot now exists during the start checkpoint. Refuted (already handled): With --create-slot, a SIGKILL during the copy phase already leaves the permanent slot today, and a rerun already fails on it. The patch adds only the checkpoint window to that exposure, which messages #3 and #4 weighed and accepted as a small added risk.
- `maintenance-fit-2` Back-patch scope and new exit-time slot drop behaviour are undefined. Refuted (no failure scenario): Before the patch, a failure before streaming started left no slot. The handler drops only the newly created slot in that same window, so the outcome is unchanged. The unstated target branches and the lack of committer review belong under committability.

### Review angles

- prior-attempts: 0 objection(s). No precedent found: the retrieved candidates share keywords only, and no thread message points to an earlier discussion of this idea. Without a prior rejection reason to carry over, the prior-attempts angle has no objection.
- standards-compat: 0 objection(s). From the standards and compatibility angle this is a client-side bug fix with no new syntax, option, GUC, catalog object or dump content. The only user-visible changes are that the slot exists earlier and is dropped on early exit, and both move existing behaviour toward what users already expect. Two points cannot be checked without the code. The first is how --slot and --create-slot interact with the new atexit drop. The second is whether the new order works against older servers that pg_basebackup still supports. A committer would likely ask about both before any back-patch to v16 through v18.
- replication-ondisk: 1 objection(s). The default temporary slot is dropped on disconnect, and the patch changes no WAL, catalog, protocol or on-disk format. The only persistent-state risk is the permanent slot from --create-slot.
- security: 0 objection(s). The proposal changes no privilege rules. pg_basebackup already needs a role with REPLICATION, and that role can create and keep slots today. Moving slot creation earlier only makes the slot hold WAL somewhat longer, and the atexit handler drops only the slot it created. A user without replication rights gains no new action and sees no new data. Not checked: what the atexit handler does when the connection is lost, which needs the code.
- maintenance-fit: 2 objection(s). The bug is real and the change is confined to pg_basebackup, with no hot-path cost for the server. The objections concern where the fix lives and its back-patch scope.
- already-exists: 0 objection(s). Workarounds exist: run pg_create_physical_replication_slot('name', true) before the backup and pass --slot, or set wal_keep_size. Neither fixes the race in the default --wal-method=stream path, and this is a bug fix in that default path rather than a new feature, so they do not make the change redundant.
- demand: 0 objection(s). This is a bug fix for a reported failure: concurrent pg_basebackup runs fail with "requested WAL segment ... has already been removed", reproduced on v16 to v18 (message #1) and confirmed by Kirill Reshke (message #2). Its costs, such as the slot living longer, fall on pg_basebackup users, so demand is not in doubt.

### Precedents retrieved

- P1: [Prevent pg_basebackup running as root](https://commitfest.postgresql.org/patch/2431/) (cf, Rejected, 2020-02-27) - judged not relevant; reason summarised from its thread
- P2: [pg_basebackup: add test about zstd compress option](https://commitfest.postgresql.org/patch/3835/) (cf, Rejected, 2022-12-06) - judged not relevant; reason summarised from its thread
- P3: [Application name for pg_basebackup, pg_receivewal and pg_recvlogical](https://commitfest.postgresql.org/patch/2335/) (cf, Rejected, 2019-11-07) - judged not relevant; reason summarised from its thread
- P4: [pg_basebackup to adjust existing data directory permissions](https://commitfest.postgresql.org/patch/2018/) (cf, Returned with feedback, 2019-11-27) - judged not relevant; reason summarised from its thread
- P5: [Revert "Prevent running pg_basebackup as root"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=dcddc3f813ff39d0e33c2b7c77092a6c0b0e9d0b) (revert, Reverted, 2020-02-07) - judged not relevant
- P6: [Revert changes to pg_basebackup and pg_waldump usage() code.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=05b5feb60e7cf36db08eb0d34dbd21854cb9e8ce) (revert, Reverted, 2017-05-17) - judged not relevant
- P7: [pg_basebackup has stricter tablespace rules than the server](https://commitfest.postgresql.org/patch/1125/) (cf, Returned with feedback, 2017-09-29) - judged not relevant
- P8: [Windows pg_basebackup unable to create >2GB pg_wal.tar tarballs ("could not close file: Invalid argument" when creating pg_wal.tar of size ~ 2^31 bytes)](https://commitfest.postgresql.org/patch/5403/) (cf, Withdrawn, 2024-11-25) - judged not relevant; reason summarised from its thread
- P9: [base backup client as auxiliary backend process](https://commitfest.postgresql.org/patch/2192/) (cf, Returned with feedback, 2020-02-28) - judged not relevant; reason summarised from its thread
- P10: [Fix base backup checksum verification for random or zero page headers](https://commitfest.postgresql.org/patch/2308/) (cf, Returned with feedback, 2020-07-30) - judged not relevant
- P11: [Reorder tablespaces for streaming basebackups](https://commitfest.postgresql.org/patch/1035/) (cf, Rejected, 2017-03-31) - judged not relevant; reason summarised from its thread
- P12: [Changing WAL Header to reduce contention during ReserveXLogInsertLocation()](https://commitfest.postgresql.org/patch/1455/) (cf, Returned with feedback, 2018-07-09) - judged not relevant; reason summarised from its thread
- P13: [Fix checksum verification in base backups for zero page headers](https://commitfest.postgresql.org/patch/2719/) (cf, Returned with feedback, 2021-09-13) - judged not relevant
- P14: [Revise documentation on base backups](https://commitfest.postgresql.org/patch/379/) (cf, Rejected, 2015-10-30) - judged not relevant
- P15: [Revert "Reduce checkpoints and WAL traffic on low activity database server"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=b8b69d89905e04b910bcd65efce1791477b45d35) (revert, Reverted, 2012-06-13) - judged not relevant
- P16: [Track wal_level drop for backup management](https://commitfest.postgresql.org/patch/2994/) (cf, Returned with feedback, 2021-10-01) - judged not relevant
- P17: [Add regexp_positions(string text, pattern text, flags text) → setof record (starts integer\[\], lengths integer\[\])](https://commitfest.postgresql.org/patch/3027/) (cf, Returned with feedback, 2021-12-03) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 13; wall time: 147 s; cost reported by the CLI: $1.13
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 6 from cache, 2 summarised now, 0 failed; quotes: 1 verified, 0 dropped; relevance pass: done, 0 of 17 kept
- Tone check: 60 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T01:07:13.137Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
