# Design pre-review: Re-read subscription state after lock in AlterSubscription

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 6972](https://commitfest.postgresql.org/patch/6972/); [thread](https://www.postgresql.org/message-id/flat/akZUpiDa1UfmzYxL%40bdtpg), 25 of 25 messages read; no code read.

## Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium). No surviving objection touches 0001 and 0002: they fix reproduced bugs, copy the existing AlterPublication() re-read, and have reviewer support. The surviving objections are fixable and concern 0003 and 0004 (demand-1, already-exists-1) and the unlocked OWNER TO path (maintenance-fit-2). The backpatch range for 0001 and 0002 is a committer's decision. Confidence is medium because only 3 of 25 messages were visible.

Not checked: Without the code, nobody can confirm that every use of subscription state in AlterSubscription() and DropSubscription() now follows the re-read. The interaction of the retry loops with lock release on a changed OID is also unchecked.

## Prior attempts

- [Fix alter subscription concurrency errors](https://www.postgresql.org/message-id/flat/HE1PR8303MB0075BF78AF1BE904050DA16BF7729@HE1PR8303MB0075.EURPRD83.prod.outlook.com) (same idea; Withdrawn, 2023-03-28). "I think the patch as-submitted is pretty uninteresting, mainly because the design of adding bespoke lock code for subscription objects doesn't scale." ([message](https://www.postgresql.org/message-id/956468.1679335244%40sss.pgh.pa.us)) Different now: 0001 and 0002 do a single re-read by OID after the lock, which [P2] did not split out. 0003 and 0004 still add subscription- and publication-specific retry loops, which is what [P2] was criticized for.
- [Revert "Avoid race condition between "GRANT role" and "DROP ROLE"".](https://www.postgresql.org/message-id/flat/CABgZEgczOFicCJoqtrH9gbYMe_BV3Hq8zzCBRcMgmU6LRsihUA@mail.gmail.com) (related; Reverted, 2025-09-16). "The race-condition problem is low severity and not worth taking risks for." Different now: [P13] was about GRANT role and DROP ROLE, a different area. Its reason bears on the open backpatch question for 0001 and 0002.

## Objections a reviewer is likely to raise

### 1. Retry loops in 0003 and 0004 answer a race no user has reported

A user drops and recreates a same-named subscription or publication while another session's ALTER waits on the lock. Once 0001 and 0002 are in, that session gets a clear error, and the user retries. 0003 and 0004 would turn that error into a silent success, at the cost of two new lookup-lock-retry loops that every later change to these DDL paths must keep correct.

The material has no field report of the drop-and-recreate or OID-reuse case. Amit Kapila calls it theoretical (messages #17, #20), and the author agrees OID reuse is theoretical (message #21). The author argues consistency: DROP PUBLICATION already retries through get_object_address(). That argument is about design, not demand. Every bug actually shown in the thread, including the orphaned pg_subscription_rel rows in message #23 and the conflict log table errors in message #1, is fixed by the single re-read in 0001 and 0002.

**Question:** Which concrete failure, seen by a user or a test, still happens with 0001 and 0002 applied and is fixed only by the retry loops in 0003 and 0004?

<sub>fixable, level 1b, demand; messages [#1](https://www.postgresql.org/message-id/akZUpiDa1UfmzYxL%40bdtpg), [#17](https://www.postgresql.org/message-id/CAA4eK1%2B%2BSdeMS2vqMMt%2BnSBC0CQ-K%2B0ESkdDdkH4SYbrxmxWLw%40mail.gmail.com), [#20](https://www.postgresql.org/message-id/CAA4eK1KKtkHr4S7uE%3DVLonXAwO4YOsZzFUQefmvbNkCBAe_chg%40mail.gmail.com), [#21](https://www.postgresql.org/message-id/akvBczTSnfia0fEP%40bdtpg), [#23](https://www.postgresql.org/message-id/CANhcyEVAqvm%2BONcghSGbq4m94PuQ2JqB4sykVQv-iLKuiEO60A%40mail.gmail.com); `demand-1`</sub>

### 2. Retry loops in 0003/0004 duplicate get_object_address() instead of reusing it

A later patch fixes missing_ok handling or error reporting in the get_object_address() retry loop. The private copies in subscriptioncmds.c and publicationcmds.c keep the old behaviour, and nothing fails when they drift apart. Users find out when DROP PUBLICATION and DROP SUBSCRIPTION handle the same concurrent case differently.

Message #21 says get_object_address() already does lookup, lock and retry for OBJECT_SUBSCRIPTION and OBJECT_PUBLICATION, and that DROP PUBLICATION uses it. 0003 and 0004 instead add two more copies of the RangeVarGetRelidExtended() loop for one object class each. An earlier subscription-only retry loop was withdrawn after reviewers called bespoke lock code for subscriptions a design that does not scale, and pointed to get_object_address() [P2]. 0001 and 0002 follow the existing AlterPublication() re-read by OID and are not affected by this objection.

Precedent: [Fix alter subscription concurrency errors](https://www.postgresql.org/message-id/flat/HE1PR8303MB0075BF78AF1BE904050DA16BF7729@HE1PR8303MB0075.EURPRD83.prod.outlook.com) (Withdrawn, 2023-03-28)

**Question:** Why can AlterSubscription(), DropSubscription() and the AlterPublication() tables/schemas branch not get their locked OID from get_object_address(), extended if needed?

<sub>fixable, level 2, design; messages [#13](https://www.postgresql.org/message-id/akfXqSGeRpL%2B2Ypz%40bdtpg), [#17](https://www.postgresql.org/message-id/CAA4eK1%2B%2BSdeMS2vqMMt%2BnSBC0CQ-K%2B0ESkdDdkH4SYbrxmxWLw%40mail.gmail.com), [#20](https://www.postgresql.org/message-id/CAA4eK1KKtkHr4S7uE%3DVLonXAwO4YOsZzFUQefmvbNkCBAe_chg%40mail.gmail.com), [#21](https://www.postgresql.org/message-id/akvBczTSnfia0fEP%40bdtpg); `already-exists-1`</sub>

### 3. Post-lock owner recheck does not cover ALTER SUBSCRIPTION OWNER TO

Session 1 runs ALTER SUBSCRIPTION SET and passes the post-lock owner recheck. Session 2 runs ALTER SUBSCRIPTION OWNER TO, which takes no subscription object lock, and commits. Session 1 then acts on an owner that is already stale, and the user still gets "tuple concurrently updated".

The v6-0001 commit message says owner changes "do not yet acquire the subscription object lock". So the new recheck in 0001 and 0002 narrows the window but does not close it. Message #1 used this same fact to argue that a recheck is pointless, and message #25 reverses that without fixing the lock protocol. That leaves one subscription DDL outside the locking rule that every later DDL must follow, and a reviewer can easily miss this.

**Question:** Should AlterSubscriptionOwner() take AccessExclusiveLock on the subscription object in this series, so that the post-lock owner recheck becomes a real guarantee?

<sub>fixable, level 2, design; messages [#1](https://www.postgresql.org/message-id/akZUpiDa1UfmzYxL%40bdtpg), [#25](https://www.postgresql.org/message-id/anWUyq2JUt4BwxRq%40bdtpg); `maintenance-fit-2`</sub>

## Questions for the author

1. Which concrete failure, seen by a user or a test, still happens with 0001 and 0002 applied and is fixed only by 0003 and 0004?
2. Why can AlterSubscription(), DropSubscription() and the AlterPublication() tables/schemas branch not get their locked OID from get_object_address(), extended if needed?
3. Should AlterSubscriptionOwner() take AccessExclusiveLock on the subscription object in this series, so that the post-lock owner recheck becomes a guarantee?
4. To which branches would 0001 and 0002 go, and what user-visible failure do they fix on branches without conflict_log_destination?
5. Which branches allow the REFRESH PUBLICATION and DROP SUBSCRIPTION race that leaves orphaned pg_subscription_rel rows, and do existing clusters need a cleanup step?

## Touch points

- **security / privileges:** 0001 and 0002 move the password_required check and add an owner recheck after the lock; the DROP hook runs later.
- **backpatchability:** The author proposes backpatching 0001 and 0002; the CommitFest entry targets stable, and a reviewer doubts it.
- **maintenance burden ("who maintains this in 10 years"):** 0003 and 0004 add two hand-written lookup, owner check and lock retry loops beside get_object_address().

## Who would commit it, and when

- Owner: qualified committer engaged. Amit Kapila takes part in the thread, supports 0001 and questions 0003, 0004 and the backpatch. The owner must know subscription DDL locking.
- Priority: proceed; audience niche; review cost low. 0001 and 0002 are small, fix reproduced races in concurrent subscription DDL and have reviewer support, so they can proceed alone. 0003 and 0004 raise the burden and can wait until the get_object_address() and demand questions have answers.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 4 refuted, the proposal restated, all 15 touch points, run details. Warnings in this run: 1.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
