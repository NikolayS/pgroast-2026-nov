# Design pre-review: WAL record CRC calculated incorrectly because of underlying buffer modification

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 7532](https://commitfest.postgresql.org/patch/7532/) (Needs review); authors: Greg Burd (gregburd)
- Thread: [WAL record CRC calculated incorrectly because of underlying buffer modification](https://www.postgresql.org/message-id/flat/055bb33c-bccc-bc1d-c2f8-8598534448ac%40gmail.com), started 2024-05-10
- Messages read: 11 of 11
- Patch prose read: xloginsert-memchk.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-COMMITTER-EYES** (confidence: low)

This is a real bug with buildfarm failures and a reproducer, so levels 1a and 1b pass. The visible material holds no fix: the only attachment is a debugging assertion, and message #10 rules out the one candidate. The surviving objections, replication-ondisk-2 and maintenance-fit-1, are fixable design points about the back-branch plan and an unenforced REGBUF_NO_CHANGE rule. The second CommitFest thread and any newer patch are not shown.

No level of the review pyramid fails.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: passed
- Levels 3-5 (implementation): out of scope

Provisional: The fix itself could not be judged: the only visible patch is the debugging assertion, and the second thread attached to the CommitFest entry is not shown. The back-branch approach and the lock cost are unknown.

## The proposal, restated

**Problem.** Buildfarm members dodo and chipmunk logged "incorrect resource manager data checksum" errors. XLogRecordAssemble() computes the record CRC over a buffer that another backend changes before the record is copied to WAL. The visible case is hashbucketcleanup() during vacuum, with wal_consistency_checking enabled.

**Proposed design.** The attached patch is a debugging aid. It copies the registered block and asserts it is unchanged after the CRC step, so it shows the bug and does not fix it. A candidate fix in the thread skips the full-page image for REGBUF_NO_CHANGE buffers even under XLR_CHECK_CONSISTENCY, but Jeff Davis later called it not acceptable.

**What the author claims.**
- The reporter reproduces the assertion failure on iterations 7, 10 and 17 of 100 runs of the hash_index test with wal_consistency_checking = all.
- A reduced psql reproducer with concurrent insert, drop and vacuum hits the same assertion.
- The reporter reproduced the issue on a 2018 commit, so it is not a recent regression.
- The thread says the candidate REGBUF_NO_CHANGE change stops the reduced reproducer from failing.

**Thread state.** Andres Freund, Thomas Munro, Jeff Davis, Michael Paquier and Robert Haas replied in May 2024. No fix is agreed: Jeff wants hash indexes to follow the locking rules or copy the page as XLogSaveBufferForHint() does, and nobody has answered why hash indexes avoid the exclusive lock.

## 2. Prior attempts

No precedent found (corpus searched for items before 2024-05-10; 24 candidates retrieved, none is the same or a similar idea).

## 3. Objections a reviewer is likely to raise

### 1. No back-branch fix that keeps hash WAL records replayable

fixable · level 2, rubric item 6 · backpatchability, WAL / replication / standby · `replication-ondisk-2`

An operator applies a minor release to the primary before the standby, as the upgrade notes advise against but as rolling upgrades often do. If the fix changes which blocks a hash record registers or how it registers them, the older standby fails to replay, or replays differently, and the operator finds out when replication stops.

Message #6 reproduces the bug on a 2018 commit, so every supported branch needs a fix. Message #5 notes that the back branches lack REGBUF_NO_CHANGE, and message #7 offers only two untested ideas. Message #10 rules out the one candidate fix, and the fixes Jeff Davis suggests (exclusive locking, or copying the page as XLogSaveBufferForHint() does) may change hash record contents or locking during replay. The CommitFest entry adds a second thread, but the material does not show it.

Thread: [#5 (Thomas Munro, 2024-05-11)](https://www.postgresql.org/message-id/CA%2BhUKGJEtb65B%2Bv2Vk25WbvvALH5DxL5Oye%3DDuhCVP8K1X3Obg%40mail.gmail.com), [#6 (Alexander Lakhin, 2024-05-11)](https://www.postgresql.org/message-id/35e2c2f8-f65a-f93a-ea23-957bb67a2d42%40gmail.com), [#7 (Thomas Munro, 2024-05-12)](https://www.postgresql.org/message-id/CA%2BhUKG%2B%3Dcb86CYa4W42z4wFBMwjQE2%3DO9RFC%2Bi4QZuCB%2Bd2p0A%40mail.gmail.com), [#10 (Jeff Davis, 2024-05-17)](https://www.postgresql.org/message-id/a4e1196a77f154ccf8df486a632d998525a1becc.camel%40j-davis.com)

**Question:** Which fix goes into the back branches, and does it leave the hash WAL record format unchanged, so that standbys on older and newer minor releases replay the same records?

Counter-check: held. Message #6 reproduces the bug on a 2018 commit, and the visible material has no fix for any branch, with message #7 offering only two untried ideas. The mixed-minor-version scenario is speculative, but the missing back-branch plan is real.

### 2. A fix that keeps REGBUF_NO_CHANGE leaves its rules unenforced

fixable · level 2, rubric item 9 · maintenance burden ("who maintains this in 10 years"), WAL / replication / standby · `maintenance-fit-1`

A later hash or extension change registers a pinned, unlocked buffer with REGBUF_NO_CHANGE but without REGBUF_NO_IMAGE. Nothing fails at insert time. Months later a buildfarm member running wal_consistency_checking, or a user standby, logs "incorrect resource manager data checksum", and someone again has to trace it back.

Message #8 names an invariant (REGBUF_NO_CHANGE implies REGBUF_NO_IMAGE) that no code checks, and says _hash_freeovflpage() already breaks it. Message #10 rules out the special case in message #5. This bug survived from at least 2018 (message #6), so a silent rule is not enough protection. Messages #8 and #9 prefer removing the flag, but nobody has committed to that work.

Thread: [#5 (Thomas Munro, 2024-05-11)](https://www.postgresql.org/message-id/CA%2BhUKGJEtb65B%2Bv2Vk25WbvvALH5DxL5Oye%3DDuhCVP8K1X3Obg%40mail.gmail.com), [#6 (Alexander Lakhin, 2024-05-11)](https://www.postgresql.org/message-id/35e2c2f8-f65a-f93a-ea23-957bb67a2d42%40gmail.com), [#8 (Jeff Davis, 2024-05-16)](https://www.postgresql.org/message-id/895723aba95f108207a744c799588ec3f2fef819.camel%40j-davis.com), [#9 (Michael Paquier, 2024-05-17)](https://www.postgresql.org/message-id/ZkavFZeGgPlaQ-W5%40paquier.xyz), [#10 (Jeff Davis, 2024-05-17)](https://www.postgresql.org/message-id/a4e1196a77f154ccf8df486a632d998525a1becc.camel%40j-davis.com)

**Question:** Will the fix remove REGBUF_NO_CHANGE, or add a check that fails loudly when a caller registers an unlocked buffer that can still produce an image?

Counter-check: held. No code checks that an unlocked registered buffer can never produce an image, and messages #8 and #9 agree the flag is a weak point. The _hash_freeovflpage() case holds an exclusive lock, so it shows the missing check, not a second bug.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | no |  |
| WAL / replication / standby | **yes** | The bug is a wrong WAL record CRC, and any fix changes how hash code registers buffers for WAL. |
| wire protocol | no |  |
| catalog changes | no |  |
| ABI / extension API | **yes** | Removing or restricting REGBUF_NO_CHANGE changes the XLogRegisterBuffer() flags that extensions can use. |
| SQL-standard conflict or non-standard syntax | no |  |
| new GUC | no |  |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | no |  |
| security / privileges | no |  |
| backpatchability | **yes** | Message #6 reproduces the bug on a 2018 commit, and the back branches lack REGBUF_NO_CHANGE. |
| maintenance burden ("who maintains this in 10 years") | **yes** | No code checks that an unlocked registered buffer cannot produce a full-page image. |
| performance on the hot path | **yes** | An exclusive lock or page copy in hashbucketcleanup() adds cost to hash index vacuum, and nobody has measured it. |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. Which fix does the current patch implement: an exclusive lock on the bucket page, a page copy as XLogSaveBufferForHint() does, or something else?
2. Which fix goes into the back branches, which lack REGBUF_NO_CHANGE, and does it leave the hash WAL record format unchanged?
3. Will the fix remove REGBUF_NO_CHANGE, or add a check that fails loudly when a caller registers an unlocked buffer that can still produce an image?
4. What does the chosen fix cost in hashbucketcleanup() during vacuum of a hash index under concurrent inserts?
5. Why does hashbucketcleanup() avoid the exclusive lock on the bucket page, and does the fix keep whatever that choice protects?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** qualified committer engaged. Several committers who know WAL insertion replied in May 2024, and Jeff Davis proposed fix directions. The fix also needs someone who knows the hash index locking protocol, and message #11 leaves that question unanswered.
- **Liability:** The owner signs for a locking or WAL registration change in hash index vacuum on every supported branch, including replay behaviour on standbys, for the life of those branches.
- **Author follow-up:** Greg Burd created the CommitFest entry on 2026-10-27 and attached a second thread on 2026-11-03, after which CFbot reported the patch no longer needs a rebase. That thread is not shown.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** proceed
- **Audience:** niche
- **Maintenance burden:** low
- **Review cost:** medium
- **Why:** The bad CRC is shown only for hash indexes with wal_consistency_checking enabled, but it is a real bug that causes buildfarm failures. Review cost is a guess because no fix patch was read; the change is likely small but touches WAL insertion and hash locking on all branches.

## Appendix

### Refuted in the counter-check

- `replication-ondisk-1` The proposal does not show the bad CRC is limited to wal_consistency_checking. Refuted (no failure scenario): The scenario needs an unlocked buffer registered without REGBUF_NO_IMAGE. In _hash_freeovflpage() the caller holds an exclusive content lock on wbuf, so no other backend can change it during the copy, and hashbucketcleanup() passes REGBUF_NO_IMAGE.
- `maintenance-fit-2` Area has no engaged owner for hash index locking. Refuted (no failure scenario): The scenario assumes an unnamed regression from a fix nobody has written, while five committers replied and Jeff Davis proposed directions. Ownership belongs in the committability field, and the back-branch part is a duplicate of replication-ondisk-2.

### Review angles

- prior-attempts: 0 objection(s). No precedent found. Message #7 and message #8 link the earlier thread that added REGBUF_NO_CHANGE, and message #10 already rejected the in-thread candidate fix, so there is no prior rejection reason left to raise against any visible proposal.
- standards-compat: 0 objection(s). No user-facing contract changes. The attached patch only adds an assertion in xloginsert.c, and the thread adds no SQL syntax, GUC, catalog entry, pg_dump output or psql behaviour. The one compatibility topic is back branches, which lack REGBUF_NO_CHANGE (message #7). That depends on a fix that is not yet shown, and message #10 already withdrew the candidate fix, so there is no design to object to. If a fix changes hash WAL record contents in minor releases, a committer would likely ask how a standby on an older minor release replays them.
- replication-ondisk: 2 objection(s). Both objections are about the WAL safety of a fix that the material does not show yet. The attached assertion patch is a debugging aid and does not change on-disk or WAL state.
- security: 0 objection(s). This proposal does not touch privileges, authentication, or data exposure. The attached patch only adds an assertion to xloginsert.c, and the bad CRC appears only with wal_consistency_checking, which only a superuser can set. An ordinary user can already run insert and vacuum on a hash index, and nothing in the thread gives that user new reach. Whether the _hash_freeovflpage() case raised in message #8 can corrupt WAL without wal_consistency_checking is a correctness question for another angle.
- maintenance-fit: 2 objection(s). This is a real bug fix, and fixing it belongs in core. The only visible patch is a debugging assertion, so the objections apply to the fix direction and not to an idea that should be rejected.
- already-exists: 0 objection(s). This entry fixes a bug where hash index WAL records get a wrong CRC, and no existing feature, extension or setting prevents it. The one existing mechanism that applies, copying the page as XLogSaveBufferForHint() does, is already suggested in message #8 and message #10 as a fix, so it gives no reason to reject the entry.
- demand: 0 objection(s). This is a bug report with buildfarm failures on dodo and chipmunk and a reproducer from message #1 and #4, so demand is not in question. Corrupt WAL CRCs hurt anyone who uses hash indexes with wal_consistency_checking, and the open questions are about the fix design, which other angles cover.

### Precedents retrieved

- P1: [pageinspect: Hash index support](https://commitfest.postgresql.org/patch/758/) (cf, Returned with feedback, 2016-12-20) - judged not relevant
- P2: [Non-volatile WAL buffer](https://commitfest.postgresql.org/patch/2615/) (cf, Withdrawn, 2021-06-18) - judged not relevant; reason summarised from its thread
- P3: [Track Oldest Initialized WAL Buffer Page](https://commitfest.postgresql.org/patch/4210/) (cf, Returned with feedback, 2023-08-03) - judged not relevant; reason summarised from its thread
- P4: [Why enable_hashjoin Completely disables HashJoin](https://commitfest.postgresql.org/patch/4265/) (cf, Withdrawn, 2023-04-05) - judged not relevant; reason summarised from its thread
- P5: [Check consistency of GUC defaults between .sample.conf and pg_settings.boot_val](https://commitfest.postgresql.org/patch/3736/) (cf, Returned with feedback, 2024-01-26) - judged not relevant; reason summarised from its thread
- P6: [Logical insert/update/delete WAL records for custom table AMs](https://commitfest.postgresql.org/patch/3394/) (cf, Withdrawn, 2022-07-20) - judged not relevant; reason summarised from its thread
- P7: [Add MAIN_RELATION_CLEANUP and TOAST_TABLE_CLEANUP options to VACUUM](https://commitfest.postgresql.org/patch/2426/) (cf, Returned with feedback, 2020-09-17) - judged not relevant; reason summarised from its thread
- P8: [SLRU data page checksums](https://commitfest.postgresql.org/patch/1682/) (cf, Returned with feedback, 2018-10-02) - judged not relevant; reason summarised from its thread
- P9: [Map WAL segment files on PMEM as WAL buffers](https://commitfest.postgresql.org/patch/3181/) (cf, Returned with feedback, 2022-08-01) - judged not relevant; reason summarised from its thread
- P10: [Enabling data checksums in an online cluster](https://commitfest.postgresql.org/patch/2611/) (cf, Withdrawn, 2021-03-09) - judged not relevant
- P11: [Start WAL streaming as early as possible, without waiting for startup process to finish replaying existing WAL. This helps synchronous replication to resume quickly when significant replay lag has been built up.](https://commitfest.postgresql.org/patch/2676/) (cf, Returned with feedback, 2020-12-01) - judged not relevant
- P12: [Change a constraint's index - ALTER TABLE ... ALTER CONSTRAINT ... USING INDEX ...](https://commitfest.postgresql.org/patch/2635/) (cf, Returned with feedback, 2020-09-30) - judged not relevant
- P13: [Fix lock contention for HASHHDR.mutex](https://commitfest.postgresql.org/patch/442/) (cf, Returned with feedback, 2016-02-08) - judged not relevant
- P14: [Cygwin cleanup](https://commitfest.postgresql.org/patch/3781/) (cf, Withdrawn, 2023-03-02) - judged not relevant
- P15: [Compensate for full_page_writes when spreading checkpoint I/O](https://commitfest.postgresql.org/patch/283/) (cf, Returned with feedback, 2015-08-25) - judged not relevant
- P16: [Revert changes in HOT handling of BRIN indexes](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=e3fcca0d0d2414f3a50d6fd40eddf48b7df81475) (revert, Reverted, 2022-06-16) - judged not relevant
- P17: [Use tuple-level SIREAD locks for index-only scans](https://commitfest.postgresql.org/patch/3163/) (cf, Returned with feedback, 2021-12-02) - judged not relevant
- P18: [Flexible configuration for full-text search](https://commitfest.postgresql.org/patch/1366/) (cf, Returned with feedback, 2017-11-30) - judged not relevant
- P19: [Revert unintentional change in increasing usage count during pin of buffers, this makes buffer access strategy have no effect.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=9cf6033281fdaf938b826545a643f951086d8671) (revert, Reverted, 2017-03-20) - judged not relevant
- P20: [Move SLRU data into the regular buffer pool](https://commitfest.postgresql.org/patch/3514/) (cf, Returned with feedback, 2023-09-04) - judged not relevant
- P21: [Use fadvise in wal replay](https://commitfest.postgresql.org/patch/3707/) (cf, Returned with feedback, 2023-04-09) - judged not relevant
- P22: [Flexible configuration for full-text search](https://commitfest.postgresql.org/patch/1421/) (cf, Returned with feedback, 2019-02-04) - judged not relevant
- P23: [Allow pg_verify_checksums to specify a database to scan](https://commitfest.postgresql.org/patch/1782/) (cf, Returned with feedback, 2018-10-02) - judged not relevant
- P24: [Improve amcheck to check UNIQUE constraint in btree index](https://commitfest.postgresql.org/patch/2976/) (cf, Returned with feedback, 2021-04-08) - judged not relevant

### Warnings

- CommitFest entry has 2 threads; only the earliest is read (https://www.postgresql.org/message-id/flat/055bb33c-bccc-bc1d-c2f8-8598534448ac%40gmail.com). Others: https://www.postgresql.org/message-id/flat/auNs8b_vS2OPdTL8%40floki

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 12; wall time: 102 s; cost reported by the CLI: $1.42
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 7 from cache, 1 summarised now, 0 failed; quotes: 1 verified, 0 dropped; relevance pass: done, 0 of 24 kept
- Tone check: 62 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T04:06:05.171Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
