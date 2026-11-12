# Design pre-review: Re-read subscription state after lock in AlterSubscription

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 6972](https://commitfest.postgresql.org/patch/6972/) (Needs review); authors: Bertrand Drouvot (bertranddrouvot.pg)
- Thread: [Re-read subscription state after lock in AlterSubscription](https://www.postgresql.org/message-id/flat/akZUpiDa1UfmzYxL%40bdtpg), started 2026-07-02
- Messages read: 25 of 25
- Patch prose read: v6-0001-Re-read-subscription-state-after-lock-in-AlterSub.patch, v6-0002-Re-read-subscription-state-after-lock-in-DropSubs.patch, v6-0003-Add-invalidation-based-retry-loop-for-Alter-Drop-.patch, v6-0004-Add-invalidation-based-retry-loop-for-AlterPublic.patch, v1-0001-Re-read-subscription-state-after-lock-in-AlterSub.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium)

No surviving objection touches 0001 and 0002: they fix reproduced bugs, copy the existing AlterPublication() re-read, and have reviewer support. The surviving objections are fixable and concern 0003 and 0004 (demand-1, already-exists-1) and the unlocked OWNER TO path (maintenance-fit-2). The backpatch range for 0001 and 0002 is a committer's decision. Confidence is medium because only 3 of 25 messages were visible.

No level of the review pyramid fails.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: passed
- Levels 3-5 (implementation): out of scope

Provisional: Without the code, nobody can confirm that every use of subscription state in AlterSubscription() and DropSubscription() now follows the re-read. The interaction of the retry loops with lock release on a changed OID is also unchecked.

## The proposal, restated

**Problem.** AlterSubscription() reads the subscription catalog row before it takes AccessExclusiveLock, so a concurrent ALTER or DROP leaves it acting on stale data. Before a5918fddf10 this mostly ended in a "tuple concurrently updated" elog(), but conflict_log_destination = 'table' now creates or drops a table from stale state. A reviewer also showed REFRESH PUBLICATION racing DROP SUBSCRIPTION and leaving orphaned pg_subscription_rel rows.

**Proposed design.** 0001 and 0002 move GetSubscription() and the password_required check after LockSharedObject() in AlterSubscription() and DropSubscription(), re-read the row by OID, and recheck ownership. 0002 also honors missing_ok and runs the DROP hook only after the re-read. 0003 and 0004 add a retry loop like RangeVarGetRelidExtended() that does name lookup, owner check and locking together, for subscriptions and for the ALTER PUBLICATION tables/schemas branch.

**User-visible surface.**
- Concurrent ALTER or DROP SUBSCRIPTION reports "must be owner", "does not exist" or password_required errors instead of "tuple concurrently updated".
- DROP SUBSCRIPTION IF EXISTS no longer fails when the subscription is dropped concurrently.
- ALTER SUBSCRIPTION ... SET (conflict_log_destination = 'table') no longer fails with "relation already exists" under concurrency.
- Object access DROP hook for subscriptions now runs after the lock and existence check.
- New isolation tests subscription-owner-locking and publication-owner-locking.

**What the author claims.**
- The bug exists on all supported branches but became more visible after a5918fddf10.
- 0001 and 0002 are small and the replaced messages are elog() ones, so the author proposes backpatching them.
- 0003 and 0004 should go to master only.
- Without a re-read, a reused ctid after vacuum could in theory corrupt a tuple; the author calls this extremely rare.
- v5 fixes the orphaned pg_subscription_rel rows that Shlok Kyal reported.

**Thread state.** Dilip Kumar, Hayato Kuroda, Zhijie Hou and Amit Kapila support 0001; Shlok Kyal approved 0001 and reviewed 0002, which v6 fixed. Amit Kapila questions 0003 and 0004 as unneeded machinery for a theoretical OID-reuse risk and doubts backpatching; neither point is settled.

## 2. Prior attempts

### Fix alter subscription concurrency errors (same idea)

- **Link:** [https://commitfest.postgresql.org/patch/3842/](https://commitfest.postgresql.org/patch/3842/)
- **Thread:** [https://www.postgresql.org/message-id/flat/HE1PR8303MB0075BF78AF1BE904050DA16BF7729@HE1PR8303MB0075.EURPRD83.prod.outlook.com](https://www.postgresql.org/message-id/flat/HE1PR8303MB0075BF78AF1BE904050DA16BF7729@HE1PR8303MB0075.EURPRD83.prod.outlook.com)
- **Date:** decided 2023-03-28; started 2022-08-25
- **Outcome:** Withdrawn (cf)
- **Who objected:** Amit Kapila, Alvaro Herrera, Tom Lane (senders of the linked thread, picked by the AI summary)
- **Why it was not accepted (AI summary of the linked thread; check it there):** Reviewers objected that adding lock-and-retry lookup code specific to subscriptions does not scale, since the same problem affects other object types. Alvaro Herrera suggested using get_object_address() instead, and Tom Lane added that this might first require improving its error reporting. The author did not post an updated version for several commitfests. He then said his own test-flakiness problem had been worked around and that he was not motivated to address the issue for all DDL commands, and the entry was withdrawn.
- **Reason (quote):** "I think the patch as-submitted is pretty uninteresting, mainly because the design of adding bespoke lock code for subscription objects doesn't scale." ([message](https://www.postgresql.org/message-id/956468.1679335244%40sss.pgh.pa.us))
- **What is different this time:** 0001 and 0002 do a single re-read by OID after the lock, which [P2] did not split out. 0003 and 0004 still add subscription- and publication-specific retry loops, which is what [P2] was criticized for.

### Revert "Avoid race condition between "GRANT role" and "DROP ROLE"". (related)

- **Link:** [https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=8abbbbae610cd62faad4f9ec18e7cfa908593796](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=8abbbbae610cd62faad4f9ec18e7cfa908593796)
- **Thread:** [https://www.postgresql.org/message-id/flat/CABgZEgczOFicCJoqtrH9gbYMe_BV3Hq8zzCBRcMgmU6LRsihUA@mail.gmail.com](https://www.postgresql.org/message-id/flat/CABgZEgczOFicCJoqtrH9gbYMe_BV3Hq8zzCBRcMgmU6LRsihUA@mail.gmail.com), [https://www.postgresql.org/message-id/flat/CAM6Zo8woa62ZFHtMKox6a4jb8qQ=w87R2L0K8347iE-juQL2EA@mail.gmail.com](https://www.postgresql.org/message-id/flat/CAM6Zo8woa62ZFHtMKox6a4jb8qQ=w87R2L0K8347iE-juQL2EA@mail.gmail.com)
- **Date:** decided 2025-09-16; started 2025-02-21
- **Outcome:** Reverted (revert)
- **Who objected:** not recorded in the corpus
- **Why it was not accepted (as recorded in the corpus):** This reverts commit 98fc31d6499163a0a781aa6f13582a07f09cd7c6. That change allowed DROP OWNED BY to drop grants of the target role to other roles, arguing that nobody would need those privileges anymore. But that's not so: if you're not superuser, you still need admin privilege on the target role so you can drop it. It's not clear whether or how the dependency-based approach to solving the original problem can be adapted to keep these grants. Since v18 release is fast approaching, the sanest thing to do seems to be to revert this patch for now. The race-condition problem is low severity and not worth taking risks for. I didn't force a catversion bump in 98fc31d64, so I won't do so here either.
- **Reason (quote):** "The race-condition problem is low severity and not worth taking risks for."
- **What is different this time:** [P13] was about GRANT role and DROP ROLE, a different area. Its reason bears on the open backpatch question for 0001 and 0002.


## 3. Objections a reviewer is likely to raise

### 1. Retry loops in 0003 and 0004 answer a race no user has reported

fixable · level 1b, rubric item 3 · demand, maintenance burden ("who maintains this in 10 years") · `demand-1`

A user drops and recreates a same-named subscription or publication while another session's ALTER waits on the lock. Once 0001 and 0002 are in, that session gets a clear error, and the user retries. 0003 and 0004 would turn that error into a silent success, at the cost of two new lookup-lock-retry loops that every later change to these DDL paths must keep correct.

The material has no field report of the drop-and-recreate or OID-reuse case. Amit Kapila calls it theoretical (messages #17, #20), and the author agrees OID reuse is theoretical (message #21). The author argues consistency: DROP PUBLICATION already retries through get_object_address(). That argument is about design, not demand. Every bug actually shown in the thread, including the orphaned pg_subscription_rel rows in message #23 and the conflict log table errors in message #1, is fixed by the single re-read in 0001 and 0002.

Thread: [#1 (Bertrand Drouvot, 2026-07-02)](https://www.postgresql.org/message-id/akZUpiDa1UfmzYxL%40bdtpg), [#17 (Amit Kapila, 2026-07-06)](https://www.postgresql.org/message-id/CAA4eK1%2B%2BSdeMS2vqMMt%2BnSBC0CQ-K%2B0ESkdDdkH4SYbrxmxWLw%40mail.gmail.com), [#20 (Amit Kapila, 2026-07-06)](https://www.postgresql.org/message-id/CAA4eK1KKtkHr4S7uE%3DVLonXAwO4YOsZzFUQefmvbNkCBAe_chg%40mail.gmail.com), [#21 (Bertrand Drouvot, 2026-07-06)](https://www.postgresql.org/message-id/akvBczTSnfia0fEP%40bdtpg), [#23 (Shlok Kyal, 2026-08-05)](https://www.postgresql.org/message-id/CANhcyEVAqvm%2BONcghSGbq4m94PuQ2JqB4sykVQv-iLKuiEO60A%40mail.gmail.com)

**Question:** Which concrete failure, seen by a user or a test, still happens with 0001 and 0002 applied and is fixed only by the retry loops in 0003 and 0004?

Counter-check: held. Message #21 cites 4240e429d0c and b3ad5d02c9c as precedent for retrying on drop-and-recreate, but shows no failure that remains after 0001 and 0002. Amit Kapila's objection in messages #17 and #20 has no reviewer answer.

### 2. Retry loops in 0003/0004 duplicate get_object_address() instead of reusing it

fixable · level 2, rubric item 12 · design, maintenance burden ("who maintains this in 10 years") · `already-exists-1`

A later patch fixes missing_ok handling or error reporting in the get_object_address() retry loop. The private copies in subscriptioncmds.c and publicationcmds.c keep the old behaviour, and nothing fails when they drift apart. Users find out when DROP PUBLICATION and DROP SUBSCRIPTION handle the same concurrent case differently.

Message #21 says get_object_address() already does lookup, lock and retry for OBJECT_SUBSCRIPTION and OBJECT_PUBLICATION, and that DROP PUBLICATION uses it. 0003 and 0004 instead add two more copies of the RangeVarGetRelidExtended() loop for one object class each. An earlier subscription-only retry loop was withdrawn after reviewers called bespoke lock code for subscriptions a design that does not scale, and pointed to get_object_address() [P2]. 0001 and 0002 follow the existing AlterPublication() re-read by OID and are not affected by this objection.

**Precedents.**
- [Fix alter subscription concurrency errors](https://commitfest.postgresql.org/patch/3842/) (Withdrawn, 2023-03-28); thread: [link](https://www.postgresql.org/message-id/flat/HE1PR8303MB0075BF78AF1BE904050DA16BF7729@HE1PR8303MB0075.EURPRD83.prod.outlook.com)

Thread: [#13 (Bertrand Drouvot, 2026-07-03)](https://www.postgresql.org/message-id/akfXqSGeRpL%2B2Ypz%40bdtpg), [#17 (Amit Kapila, 2026-07-06)](https://www.postgresql.org/message-id/CAA4eK1%2B%2BSdeMS2vqMMt%2BnSBC0CQ-K%2B0ESkdDdkH4SYbrxmxWLw%40mail.gmail.com), [#20 (Amit Kapila, 2026-07-06)](https://www.postgresql.org/message-id/CAA4eK1KKtkHr4S7uE%3DVLonXAwO4YOsZzFUQefmvbNkCBAe_chg%40mail.gmail.com), [#21 (Bertrand Drouvot, 2026-07-06)](https://www.postgresql.org/message-id/akvBczTSnfia0fEP%40bdtpg)

**Question:** Why can AlterSubscription(), DropSubscription() and the AlterPublication() tables/schemas branch not get their locked OID from get_object_address(), extended if needed?

Counter-check: held. Message #21 confirms get_object_address() already retries for publications and subscriptions, and the thread does not say why 0003 and 0004 cannot use it. [P2] fits: the same subscription-only loop drew the same suggestion.

### 3. Post-lock owner recheck does not cover ALTER SUBSCRIPTION OWNER TO

fixable · level 2, rubric item 9 · design, maintenance burden ("who maintains this in 10 years"), security / privileges · `maintenance-fit-2`

Session 1 runs ALTER SUBSCRIPTION SET and passes the post-lock owner recheck. Session 2 runs ALTER SUBSCRIPTION OWNER TO, which takes no subscription object lock, and commits. Session 1 then acts on an owner that is already stale, and the user still gets "tuple concurrently updated".

The v6-0001 commit message says owner changes "do not yet acquire the subscription object lock". So the new recheck in 0001 and 0002 narrows the window but does not close it. Message #1 used this same fact to argue that a recheck is pointless, and message #25 reverses that without fixing the lock protocol. That leaves one subscription DDL outside the locking rule that every later DDL must follow, and a reviewer can easily miss this.

Thread: [#1 (Bertrand Drouvot, 2026-07-02)](https://www.postgresql.org/message-id/akZUpiDa1UfmzYxL%40bdtpg), [#25 (Bertrand Drouvot, 2026-08-07)](https://www.postgresql.org/message-id/anWUyq2JUt4BwxRq%40bdtpg)

**Question:** Should AlterSubscriptionOwner() take AccessExclusiveLock on the subscription object in this series, so that the post-lock owner recheck becomes a real guarantee?

Counter-check: held. The v6-0001 commit message says owner changes "do not yet acquire the subscription object lock", so the recheck leaves a window. The gap exists before the patch, but v6 reverses message #1 without saying whether AlterSubscriptionOwner() should take the lock.

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
| pg_dump / restore round-trip | no |  |
| psql UX conventions | no |  |
| security / privileges | **yes** | 0001 and 0002 move the password_required check and add an owner recheck after the lock; the DROP hook runs later. |
| backpatchability | **yes** | The author proposes backpatching 0001 and 0002; the CommitFest entry targets stable, and a reviewer doubts it. |
| maintenance burden ("who maintains this in 10 years") | **yes** | 0003 and 0004 add two hand-written lookup, owner check and lock retry loops beside get_object_address(). |
| performance on the hot path | no |  |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. Which concrete failure, seen by a user or a test, still happens with 0001 and 0002 applied and is fixed only by 0003 and 0004?
2. Why can AlterSubscription(), DropSubscription() and the AlterPublication() tables/schemas branch not get their locked OID from get_object_address(), extended if needed?
3. Should AlterSubscriptionOwner() take AccessExclusiveLock on the subscription object in this series, so that the post-lock owner recheck becomes a guarantee?
4. To which branches would 0001 and 0002 go, and what user-visible failure do they fix on branches without conflict_log_destination?
5. Which branches allow the REFRESH PUBLICATION and DROP SUBSCRIPTION race that leaves orphaned pg_subscription_rel rows, and do existing clusters need a cleanup step?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** qualified committer engaged. Amit Kapila takes part in the thread, supports 0001 and questions 0003, 0004 and the backpatch. The owner must know subscription DDL locking.
- **Liability:** For 0001 and 0002: changed error paths and hook timing in ALTER and DROP SUBSCRIPTION, on every branch that gets the backpatch. For 0003 and 0004: two retry loops that later changes to lookup and locking rules must keep in step.
- **Author follow-up:** The author posted six versions and fixed the orphaned pg_subscription_rel case a reviewer reported, which suggests he stays for fixes.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** proceed
- **Audience:** niche
- **Maintenance burden:** low
- **Review cost:** low
- **Why:** 0001 and 0002 are small, fix reproduced races in concurrent subscription DDL and have reviewer support, so they can proceed alone. 0003 and 0004 raise the burden and can wait until the get_object_address() and demand questions have answers.

## Appendix

### Refuted in the counter-check

- `prior-attempts-1` Retry loops in 0003 and 0004 repeat a design rejected earlier. Refuted (not specific): Duplicate of already-exists-1, which makes the same get_object_address() point and also states that 0001 and 0002 are not affected.
- `standards-compat-1` Subscription-only retry loop duplicates get_object_address() for one object type. Refuted (not specific): Duplicate of already-exists-1.
- `security-1` Owner recheck after lock does not pin ownership while OWNER TO skips the lock. Refuted (not specific): Duplicate of maintenance-fit-2. This version also names no step that acts with the old owner's authority, and the unlocked OWNER TO path exists before the patch.
- `maintenance-fit-1` Hand-written retry loops per object type in 0003 and 0004. Refuted (not specific): Duplicate of already-exists-1.

### Review angles

- prior-attempts: 1 objection(s). 0001 and 0002 differ from [P2] by doing a plain re-read by OID after the lock, and the precedents raise no objection against them.
- standards-compat: 1 objection(s). From the compatibility angle, 0001 and 0002 only replace elog() errors with proper ereport() errors and make DROP SUBSCRIPTION IF EXISTS honor missing_ok. They add no syntax, GUC or dump change, and existing scripts do not regress.
- replication-ondisk: 0 objection(s). The patches change only lock ordering and the timing of catalog reads, so they add no on-disk format, WAL record, wire protocol, catalog column or exported ABI. One open point remains: orphaned pg_subscription_rel rows already created by the REFRESH PUBLICATION and DROP race in message #23 stay in existing clusters, and the proposal does not say whether release notes or a cleanup query should cover them.
- security: 1 objection(s). There is one objection. The patch otherwise tightens the privilege checks, and the new password_required and owner rechecks do not give non-superusers anything they could not do before.
- maintenance-fit: 2 objection(s). 0001 and 0002 move existing reads after the lock and add no hot-path cost. Only the scope of 0003 and 0004 and the incomplete lock protocol raise maintenance concerns.
- already-exists: 1 objection(s). 0001 and 0002 copy the existing AlterPublication() pattern and fix a real bug; no existing feature or extension already does this, so nothing is raised against them from this angle.
- demand: 1 objection(s). 0001 and 0002 fix reproduced bugs and have broad reviewer support, so demand for them is shown. The objection covers only 0003 and 0004.

### Precedents retrieved

- P1: [Parallel tests publication and subscription might fail due to concurrent tuple update](https://commitfest.postgresql.org/patch/5459/) (cf, Rejected, 2025-03-27) - judged not relevant; reason summarised from its thread
- P2: [Fix alter subscription concurrency errors](https://commitfest.postgresql.org/patch/3842/) (cf, Withdrawn, 2023-03-28) - used in the review; reason summarised from its thread
- P3: [Allow reads to proceed during FK/trigger drops by reducing relation-level lock from AccessExclusive to ShareRowExclusive](https://commitfest.postgresql.org/patch/6124/) (cf, Withdrawn, 2025-11-25) - judged not relevant; reason summarised from its thread
- P4: [Revert: Allow locking updated tuples in tuple_update() and tuple_delete()](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=193e6d18e553a104d315ff81892d509d89a30fd8) (revert, Reverted, 2024-04-11) - judged not relevant
- P5: [DROP INDEX CONCURRENTLY on partitioned index](https://commitfest.postgresql.org/patch/2805/) (cf, Withdrawn, 2021-07-24) - judged not relevant; reason summarised from its thread
- P6: [Revert changes to CONCURRENTLY that "sped up" Xmin advance](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=e28bb885196916b0a3d898ae4f2be0e38108d81b) (revert, Reverted, 2022-05-31) - judged not relevant
- P7: [add subscription TAP test option "vcregress subscriptioncheck" for MSVC builds](https://commitfest.postgresql.org/patch/3354/) (cf, Returned with feedback, 2022-02-07) - judged not relevant; reason summarised from its thread
- P8: [Subscription code improvements](https://commitfest.postgresql.org/patch/1248/) (cf, Returned with feedback, 2018-04-07) - used in the review; reason summarised from its thread
- P9: [Revert tests for drop index concurrently.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=5ad72cee7ed3ac0016c716f0408811274d822f14) (revert, Reverted, 2012-10-18) - judged not relevant
- P10: [Revert 019_replslot_limit.pl related debugging aids.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=3f8148c256e067dc2e8929ed174671ba7dc3339c) (revert, Reverted, 2022-07-05) - judged not relevant
- P11: [Reduce lock level for ALTER TABLE ... ADD CHECK](https://commitfest.postgresql.org/patch/3088/) (cf, Withdrawn, 2021-11-21) - judged not relevant; reason summarised from its thread
- P12: [Revert "TupleHashTable: store additional data along with tuple."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=b4a07f532b40a64fb4714a3f7ab6063435411edb) (revert, Reverted, 2025-01-13) - judged not relevant
- P13: [Revert "Avoid race condition between "GRANT role" and "DROP ROLE"".](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=8abbbbae610cd62faad4f9ec18e7cfa908593796) (revert, Reverted, 2025-09-16) - used in the review
- P14: [orphaned function](https://commitfest.postgresql.org/patch/2850/) (cf, Returned with feedback, 2021-04-08) - used in the review; reason summarised from its thread
- P15: [Lock updated tuples in tuple_update() and tuple_delete()](https://commitfest.postgresql.org/patch/4099/) (cf, Returned with feedback, 2024-10-08) - judged not relevant
- P16: [Revert: Allow table AM tuple_insert() method to return the different slot](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=8dd0bb84da7d56a9e41241b26bfbf6b79644d574) (revert, Reverted, 2024-04-11) - judged not relevant
- P17: [Revert "Skip redundant anti-wraparound vacuums"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=dd9ac7d5d80608a640bb82cffb6a805ce84cf112) (revert, Reverted, 2020-03-30) - judged not relevant
- P18: [Revert the commits related to allowing page lock to conflict among parallel group members.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=cc32ec24fdf3b9808617439d41b0ba7f3eb16b04) (revert, Reverted, 2023-07-06) - judged not relevant
- P19: [Orphaned users in PG16 and above can only be managed by Superusers](https://commitfest.postgresql.org/patch/5608/) (cf, Withdrawn, 2026-03-05) - judged not relevant
- P20: [Avoid orphaned dependencies](https://commitfest.postgresql.org/patch/3106/) (cf, Returned with feedback, 2022-08-02) - used in the review
- P21: [Revert "Test that vacuum removes tuples older than OldestXmin"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=efcbb76efe406d59c2ba8b4a09e04c01158ba575) (revert, Reverted, 2024-07-22) - judged not relevant
- P22: [Revert "Use pselect(2) not select(2), if available, to wait in postmaster's loop."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=64925603c972aa3a9f1d4c42686dba67f1a7e9d0) (revert, Reverted, 2017-04-24) - judged not relevant
- P23: [POC: Lock updated tuples in tuple_update() and tuple_delete()](https://commitfest.postgresql.org/patch/3752/) (cf, Returned with feedback, 2022-10-12) - judged not relevant
- P24: [Surround CheckRelation\[Oid\]LockedByMe() with USE_ASSERT_CHECKING](https://commitfest.postgresql.org/patch/5096/) (cf, Withdrawn, 2024-07-01) - judged not relevant

### Warnings

- synthesis said NEEDS-COMMITTER-EYES but listed a level 1b objection (demand-1); changed to NEEDS-DESIGN-DISCUSSION

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 16; wall time: 160 s; cost reported by the CLI: $3.01
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 5 from cache, 3 summarised now, 0 failed; quotes: 3 verified, 0 dropped; relevance pass: done, 5 of 24 kept
- Tone check: 86 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T03:11:30.915Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
