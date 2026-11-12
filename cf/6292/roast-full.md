# Design pre-review: Proposal to allow setting cursor options on Portals

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 6292](https://commitfest.postgresql.org/patch/6292/) (Needs review); authors: Dave Cramer (dcramer)
- Thread: [Proposal to allow setting cursor options on Portals](https://www.postgresql.org/message-id/flat/CADK3HHKe1PA1U6aB5-7tWBQ0yZGgNvY7H%3DECDD9955Pas_zx_Q%40mail.gmail.com), started 2025-12-07
- Messages read: 54 of 54
- Patch prose read: v5-0002-Add-a-fetch-direction-to-Execute-under-_pq_.curso.patch, v5-0001-Add-_pq_.cursor-protocol-extension-for-cursor-opt.patch, v5-0003-libpq-add-support-for-fetching-from-a-portal.patch, 0001-wip-holdable-portals.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium)

The goal has support in the thread, and the choice of a protocol extension over a version bump is settled. The design of v5 is not: standards-compat-1 shows that negotiating _pq_.cursor changes every Bind and Execute, and the PortalHold alternative from message #52 has no answer. standards-compat-3 and maintenance-fit-2 show that Bind with cursor options and DECLARE already behave differently. already-exists-1 questions whether the Execute fields in 0002 belong in this patch.

First failing level, from the base up: **Level 2: architecture and design**. The review stops there; higher levels are not evaluated.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: failed
- Levels 3-5 (implementation): out of scope

Provisional: Without the code, the crash fixes in 0004 and 0005 and the reported isTopLevel difference for a directional Execute could not be checked. The review saw only 3 of 54 messages in full, so later thread facts come from the panel's reports.

## The proposal, restated

**Problem.** Drivers such as pgjdbc cannot create holdable or scrollable portals over the extended query protocol. Today they must rewrite queries into DECLARE CURSOR and FETCH as SQL text, which the author calls cumbersome. A user in the thread reports Java out-of-memory errors when result sets must survive a commit, because the driver then ignores the fetch size.

**Proposed design.** A new protocol extension, _pq_.cursor, negotiated at startup, adds a mandatory Int32 cursor flags field to every Bind (SCROLL, NO SCROLL, WITH HOLD). It also adds a mandatory Int32 fetch-direction field and an Int64 count to every Execute, so a client can fetch or move as FETCH and MOVE do. libpq gets a connection parameter and new functions to use both.

**User-visible surface.**
- Protocol extension _pq_.cursor, which changes the Bind and Execute message formats on the whole connection once negotiated
- libpq connection parameter protocol_cursor
- libpq functions PQsendBindWithCursorOptions and PQprotocolCursorEnabled
- libpq functions PQsendExecutePortal and PQsendBindAndExecutePortal, with PQ_FETCH_* constants
- libpq trace output prints the new Bind and Execute fields

**What the author claims.**
- Holdable cursors at protocol level fix an oversight in the original protocol and expose existing server features.
- An all-zero Execute trailer behaves byte-for-byte like today's Execute, so drivers that use the row-count field keep working.
- Mandatory fields are safer than optional trailers detected by message length when later extensions extend the same message.
- Opt-in negotiation avoids adding 4 bytes per Bind and 12 per Execute for clients that do not use cursors.
- Everything the patch enables is already possible through SQL today.

**Thread state.** After a long dispute, Tom Lane, Robert Haas and Jacob Champion favoured a protocol extension over a version bump, and v5 follows that. Sehrope Sarkuni and Vladimir Sitnikov then reported crashes, check gaps, and pooler breakage. Vladimir proposed a separate PortalHold message so that Bind and Execute stay unchanged, and the author has not answered that proposal.

## 2. Prior attempts

### cursors with prepared statements (similar idea)

- **Link:** [https://commitfest.postgresql.org/patch/1667/](https://commitfest.postgresql.org/patch/1667/)
- **Thread:** [https://www.postgresql.org/message-id/flat/762cc764-74f0-13fb-77ed-16f91c90f40d@2ndquadrant.com](https://www.postgresql.org/message-id/flat/762cc764-74f0-13fb-77ed-16f91c90f40d@2ndquadrant.com)
- **Date:** decided 2018-12-01; started 2018-06-07
- **Outcome:** Returned with feedback (cf)
- **Who objected:** Heikki Linnakangas (senders of the linked thread, picked by the AI summary)
- **Why it was not accepted (AI summary of the linked thread; check it there):** Heikki Linnakangas objected to the USING clause on DECLARE as a non-standard extension and proposed `DECLARE c CURSOR FOR EXECUTE p (foo, bar)` instead. Robert Haas supported the author's syntax, and the discussion stopped in July 2018 with no agreement. After months without activity and no reply to a status inquiry, Dmitry Dolgov marked the entry returned with feedback.
- **Reason (quote):** "I hope it's not another abandoned patch, but due to lack of response I'm marking it as returned with feedback." ([message](https://www.postgresql.org/message-id/CA%2Bq6zcWJifdjoXiqPqiRAqw%3DEE%3DDVLksr4kFz%3DVevwyrU_iqTA%40mail.gmail.com))
- **What is different this time:** P1 added SQL syntax to declare a cursor over a prepared statement. This proposal leaves SQL alone and adds cursor options to Bind and fetch fields to Execute through a negotiated protocol extension.


## 3. Objections a reviewer is likely to raise

### 1. Negotiating _pq_.cursor changes every Bind and Execute on the connection

fixable · level 2, rubric item 12 · wire protocol, maintenance burden ("who maintains this in 10 years") · `standards-compat-1`

An application sets protocol_cursor=1 behind PgBouncer 1.22.1 with ignore_startup_parameters = _pq_.cursor. libpq gets no NegotiateProtocolVersion, assumes the option was accepted, and adds the new fields. From then on every extended query on that connection fails with "invalid message format", including queries that never use a cursor.

Message #52 reports this failure and shows that a pooler cannot share a server connection between clients that did and did not negotiate the option. In v5 the fields are mandatory once negotiated, so the cost falls on every Bind and Execute rather than only on messages that use the feature. Message #52 proposes a separate PortalHold message that leaves Bind and Execute unchanged, and the thread has no answer to it. The author's reason for mandatory fields, that length sniffing cannot carry a second extension (message #48), holds just as well for a new message.

Thread: [#48 (Dave Cramer, 2026-09-09)](https://www.postgresql.org/message-id/CADK3HH%2BpdutF6dQG9-fDPtJri%2B%3D1-o9YKJQLeO9u6gj7cisU0A%40mail.gmail.com), [#52 (Vladimir Sitnikov, 2026-09-15)](https://www.postgresql.org/message-id/CAB%3DJe-HDihfXy%2BXRQtukPkyft%3Dx2%2BC660wroyfRh7g9b4iiV5g%40mail.gmail.com), [#28 (Jelte Fennema-Nio, 2026-01-14)](https://www.postgresql.org/message-id/CAGECzQSHKR2LtBSnDT_bjdpE1%3DcVzZLRcb2k5d_K%3DBMvJCFihg%40mail.gmail.com), [#30 (Jacob Champion, 2026-01-15)](https://www.postgresql.org/message-id/CAOYmi%2BkkQqRw2ajt5e-CX5YchrMwS_qJ4%2B94Yjxh%3DdHGo%2B8jPA%40mail.gmail.com)

**Question:** Why must holdability travel inside Bind instead of in a separate message, given that only PreCommit_Portals reads CURSOR_OPT_HOLD?

Counter-check: held. Message #52 reports both the PgBouncer 1.22.1 failure and the loss of pool sharing, and the thread has no reply to the PortalHold proposal. The opt-in connection parameter limits who is affected but does not answer why Bind and Execute must change.

### 2. Protocol cursors do not follow the rules DECLARE applies

fixable · level 2, rubric item 12 · planner behaviour change, SQL-standard conflict or non-standard syntax · `standards-compat-3`

A session with plan_cache_mode = force_generic_plan binds a prepared GROUP BY query with $1 = 0. Without SCROLL it returns 10 rows. With SCROLL the replan folds the parameter in as a constant and Bind fails with division by zero. The same query through DECLARE SCROLL is planned under DECLARE's rules, so the two paths give different results.

v5 crashes on SCROLL for plans that cannot scan backward and on HOLD for non-SELECT portals (message #49). The 0004 fixes then make Bind differ from DECLARE in other ways: it accepts FOR UPDATE with a data-modifying WITH that DECLARE refuses, and it rejects FOR UPDATE in subqueries, CTEs and views that DECLARE accepts (message #51). The documentation says "as for DECLARE", so drivers will rely on a contract that is not met. The material does not show that the author has adopted 0004 or chosen which behaviour is correct.

Thread: [#49 (Sehrope Sarkuni, 2026-09-10)](https://www.postgresql.org/message-id/CAH7T-aqbVB8t1ZPiqVMhYvwU8Q3AbABBUgSaDvafAWY7xbeS3g%40mail.gmail.com), [#51 (Vladimir Sitnikov, 2026-09-15)](https://www.postgresql.org/message-id/CAB%3DJe-HjGipRGuxP98BnqO1jhTTrSJx5z7m9000ugc0%2B3F1rKg%40mail.gmail.com)

**Question:** For which queries and settings does a Bind with SCROLL or HOLD behave the same as DECLARE with the same options, and where does it differ on purpose?

Counter-check: held. Messages #49 and #51 show the crashes in v5 and the remaining differences from DECLARE after 0004. The author has not said whether 0004 is adopted or which behaviour is intended.

### 3. Second cursor planning and checking path beside DECLARE

fixable · level 2, rubric item 9 · maintenance burden ("who maintains this in 10 years"), planner behaviour change · `maintenance-fit-2`

A future patch adds a cursor restriction or planning flag to DECLARE but not to Bind. Clients that use Bind with SCROLL or HOLD then get plans or checks that differ from DECLARE, or hit an executor assertion. Nothing fails at build or test time when the two paths drift.

Message #49 found crashes because Bind reused cached plans not built for backward scans, and allowed HOLD on non-SELECT portals. The fix in 0004 re-plans outside the plan cache. Message #51 shows it still differs from DECLARE in four ways: data-modifying WITH, force_generic_plan, a stricter FOR UPDATE check, and no CURSOR_OPT_FAST_PLAN. Each cursor rule now lives in two places.

Thread: [#49 (Sehrope Sarkuni, 2026-09-10)](https://www.postgresql.org/message-id/CAH7T-aqbVB8t1ZPiqVMhYvwU8Q3AbABBUgSaDvafAWY7xbeS3g%40mail.gmail.com), [#51 (Vladimir Sitnikov, 2026-09-15)](https://www.postgresql.org/message-id/CAB%3DJe-HjGipRGuxP98BnqO1jhTTrSJx5z7m9000ugc0%2B3F1rKg%40mail.gmail.com)

**Question:** Can Bind with cursor options share DECLARE's planning and validation code, so that one rule change covers both paths?

Counter-check: held. Message #51 lists four differences from DECLARE that remain after 0004, so the two paths have already drifted. No message says whether Bind can share DECLARE's planning and checks.

### 4. Execute fetch fields duplicate SQL FETCH and MOVE on named portals

fixable · level 1b, rubric item 12 · doesn't fit what Postgres is about (extension / external tool / already solved), wire protocol, maintenance burden ("who maintains this in 10 years") · `already-exists-1`

Under _pq_.cursor, every driver and pooler must send and parse 12 extra bytes on every Execute to gain what FETCH and MOVE already do. A second fetch path then drifts from SQL FETCH. One case is known: a CALL with an INOUT parameter that commits works through a plain Execute and fails with 2D000 through a directional Execute.

SQL FETCH and MOVE already read a named portal created by Bind in any direction, as message #52 points out, and the author says in message #54 that everything the patch enables is possible through SQL today. The 0002 commit message justifies the fields because only SQL FETCH could scroll, so they save a text query and nothing more. Message #52 lists further defects that exist only because of the second path, such as direction checks running after log_statement.

Thread: [#52 (Vladimir Sitnikov, 2026-09-15)](https://www.postgresql.org/message-id/CAB%3DJe-HDihfXy%2BXRQtukPkyft%3Dx2%2BC660wroyfRh7g9b4iiV5g%40mail.gmail.com), [#54 (Dave Cramer, 2026-11-10)](https://www.postgresql.org/message-id/CADK3HHKM4%2BhTWdLbg%2BEB0pAtWh%3D7dyzrnC0EZGoCYwMWKMknNg%40mail.gmail.com), [#48 (Dave Cramer, 2026-09-09)](https://www.postgresql.org/message-id/CADK3HH%2BpdutF6dQG9-fDPtJri%2B%3D1-o9YKJQLeO9u6gj7cisU0A%40mail.gmail.com)

**Question:** Which driver task needs a directional Execute that "FETCH BACKWARD n FROM portal" sent over the extended protocol cannot do, apart from saving one query string?

Counter-check: held. A reviewer asked for the Execute fields in message #47, but the author says in message #54 that SQL can do all of it. Nobody in the thread names a task that FETCH on the named portal cannot do, and message #52 reports defects specific to 0002.

### 5. Bind-time HOLD now covers only what DECLARE WITH HOLD already does

fixable · level 1b, rubric item 2 · desirability, doesn't fit what Postgres is about (extension / external tool / already solved), wire protocol · `already-exists-2`

After 0004, a driver can set HOLD only on a single SELECT, which DECLARE ... WITH HOLD with $n parameters and binary FETCH results already supports. The case SQL cannot do, holding the result of INSERT ... RETURNING across COMMIT, is still rejected. Every Bind on the connection changes format for a gain that the driver could get with SQL.

Message #52 notes that holding a portal at the protocol level is strongest for RETURNING and data-modifying WITH, and that 0004 excludes exactly those. The author's case in messages #1 and #10 is that rewriting queries into DECLARE is cumbersome, which is a matter of convenience rather than a missing capability. The out-of-memory report in message #37 comes from pgjdbc ignoring the fetch size for holdable result sets; DECLARE WITH HOLD plus FETCH avoids that today.

Thread: [#1 (Dave Cramer, 2025-12-07)](https://www.postgresql.org/message-id/CADK3HHKe1PA1U6aB5-7tWBQ0yZGgNvY7H%3DECDD9955Pas_zx_Q%40mail.gmail.com), [#10 (Dave Cramer, 2025-12-14)](https://www.postgresql.org/message-id/CADK3HHJVX_H%2B8ge%2BvMzU35CzbUSCw08FJ_1G39m2-SLwM3V6pQ%40mail.gmail.com), [#37 (Patrick Reinhart, 2026-03-09)](https://www.postgresql.org/message-id/2b3d5da9-34ba-49ee-a052-b6dd7233c524%40gmail.com), [#49 (Sehrope Sarkuni, 2026-09-10)](https://www.postgresql.org/message-id/CAH7T-aqbVB8t1ZPiqVMhYvwU8Q3AbABBUgSaDvafAWY7xbeS3g%40mail.gmail.com), [#52 (Vladimir Sitnikov, 2026-09-15)](https://www.postgresql.org/message-id/CAB%3DJe-HDihfXy%2BXRQtukPkyft%3Dx2%2BC660wroyfRh7g9b4iiV5g%40mail.gmail.com), [#54 (Dave Cramer, 2026-11-10)](https://www.postgresql.org/message-id/CADK3HHKM4%2BhTWdLbg%2BEB0pAtWh%3D7dyzrnC0EZGoCYwMWKMknNg%40mail.gmail.com)

**Question:** What does a held SELECT portal created by Bind give pgjdbc that DECLARE ... WITH HOLD with parameters and binary FETCH cannot, enough to justify changing every Bind?

Counter-check: held. Several people support the feature, but the stated reason is that the DECLARE rewrite is cumbersome, and DECLARE WITH HOLD can do the SELECT case. The point depends on 0004, which the author has not adopted, but v5 without it crashes on non-SELECT HOLD.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | no |  |
| WAL / replication / standby | no |  |
| wire protocol | **yes** | _pq_.cursor adds a mandatory Int32 to every Bind and an Int32 plus Int64 to every Execute once negotiated. |
| catalog changes | no |  |
| ABI / extension API | **yes** | exports.txt gains four libpq functions, and libpq-fe.h gains the PQ_FETCH_* constants. |
| SQL-standard conflict or non-standard syntax | **yes** | see objection standards-compat-3: Protocol cursors do not follow the rules DECLARE applies |
| new GUC | no |  |
| planner behaviour change | **yes** | A Bind with SCROLL needs a plan that can scan backward, so it cannot reuse a cached plan as is (messages #49, #51). |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | no |  |
| security / privileges | **yes** | A held portal outlives its transaction, so behind a transaction-mode pooler another client can reach it (message #52). |
| backpatchability | no |  |
| maintenance burden ("who maintains this in 10 years") | **yes** | The patch adds a permanent wire format, four libpq exports, and a second cursor checking path beside DECLARE. |
| performance on the hot path | **yes** | exec_bind_message and the Execute path in postgres.c read extra fields, but only on connections that negotiated the option. |
| doesn't fit what Postgres is about (extension / external tool / already solved) | **yes** | The author states in message #54 that SQL DECLARE, FETCH and MOVE already provide everything the patch enables. |

## 5. Questions the author must answer before anyone reviews code

1. Why must holdability travel inside Bind, and not in a separate message such as the PortalHold proposed in message #52, which leaves Bind and Execute unchanged?
2. Which driver task needs a directional Execute that SQL FETCH or MOVE on the same named portal cannot do, apart from saving one query string?
3. Will Sehrope's 0004 and 0005 be folded into the next version, and if so, which of the remaining differences from DECLARE listed in message #51 are intended?
4. Can Bind with SCROLL or HOLD share DECLARE's planning and validation code, so that one rule change covers both paths?
5. How does a transaction-mode pooler learn that a held portal exists or is gone, or will protocol.sgml state that HOLD is unsupported there?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** silent. It needs a committer who knows the extended query protocol, portal and plan cache code in tcop, and libpq. Tom Lane, Robert Haas and Jacob Champion joined only the negotiation debate, not the v5 design.
- **Liability:** The owner signs for a wire format and four libpq exports that cannot be removed. The owner also signs for every later difference between Bind and DECLARE, and for pooler breakage reports.
- **Author follow-up:** The author posted v5 in message #48, replied in message #54 and moved the entry across CommitFests. The PortalHold proposal and the 0004 and 0005 fixes have no answer yet.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** fix design first
- **Audience:** a sizeable minority
- **Maintenance burden:** high
- **Review cost:** high
- **Why:** Demand comes from JDBC holdable result sets, postgres_fdw interest and one out-of-memory field report. The series is about 100 KiB across postgres.c, backend_startup.c and libpq, and it fixes a wire format, so the open design question from message #52 should be settled before code review.

## Appendix

### Held up in the counter-check, not listed above

- `replication-ondisk-3` (fixable, level 2) Held portals are session state that poolers cannot track. Not listed: not among the strongest objections.

### Refuted in the counter-check

- `standards-compat-2` Execute fetch fields duplicate FETCH and give Execute a second meaning. Refuted (not specific): Duplicate of already-exists-1. Its one separate point, that count 0 re-fetches the current row, copies FETCH FORWARD 0 and the libpq documentation in 0003 states it.
- `replication-ondisk-1` Negotiating _pq_.cursor changes every Bind and Execute on the connection. Refuted (not specific): Duplicate of standards-compat-1, which makes the same pool-sharing point and asks the same question about separate messages.
- `replication-ondisk-2` A dropped _pq_.cursor parameter breaks every extended query. Refuted (not specific): Duplicate of standards-compat-1, which uses the same PgBouncer 1.22.1 scenario. Treating a missing NegotiateProtocolVersion as acceptance is how `_pq_.` options are defined, so only the size of the breakage is specific to this patch.
- `security-1` Held portals become readable by other clients behind a transaction pooler. Refuted (no failure scenario): Message #52 reports that the next client's Bind of C_1 fails with 42P03, so a driver that binds before it fetches gets an error, not another tenant's rows. The pooler side of this is covered by replication-ondisk-3, and DECLARE WITH HOLD has the same exposure today.
- `maintenance-fit-1` Whole-connection Bind and Execute format change does not accumulate well. Refuted (not specific): Duplicate of standards-compat-1. The added accumulation argument rests on later extensions that the material does not show.
- `maintenance-fit-3` Execute fetch fields duplicate SQL FETCH and widen the patch. Refuted (not specific): Duplicate of already-exists-1.
- `maintenance-fit-4` HOLD limited to SELECT matches what DECLARE WITH HOLD already gives. Refuted (not specific): Duplicate of already-exists-2. Its failure scenario about poolers is covered by replication-ondisk-3.
- `demand-1` Fetch direction on Execute has no user asking for it. Refuted (not specific): Duplicate of already-exists-1. Its pooler scenario is covered by standards-compat-1.
- `demand-2` HOLD limited to a single SELECT only duplicates DECLARE WITH HOLD. Refuted (not specific): Duplicate of already-exists-2.

### Review angles

- prior-attempts: 0 objection(s). No listed precedent records a reason that applies here: P1 stalled on syntax and inactivity, and P4, P9 and P21 concern other protocol changes. The thread already replaced its own first design, a 3.3 version bump (message #1), with _pq_.cursor after messages #20, #24 and #29, so that question is settled.
- standards-compat: 3 objection(s). Objections found; the move from a version bump to a protocol extension was settled in the thread (messages #20 to #35), so it is not raised.
- replication-ondisk: 3 objection(s). The proposal changes no on-disk format, WAL or catalog. Its persistent effects are in the wire protocol and in session state for held portals, and the objections cover those.
- security: 1 objection(s). Apart from pooler session state, the extension only exposes cursor options that SQL already grants to every role, and it adds no new privilege. Sehrope's 0004 fixes the crash and overflow cases. Their correctness cannot be checked without the code.
- maintenance-fit: 4 objection(s). Objections listed. On committability, no committer has engaged with v5. Tom, Robert and Jacob took part only in the negotiation debate.
- already-exists: 2 objection(s). Both objections rest on SQL DECLARE, FETCH and MOVE, which already provide the capability. Neither says the protocol gap is imaginary.
- demand: 2 objection(s). Demand for protocol-level WITH HOLD is shown: the JDBC holdable result set API (message #1), postgres_fdw interest (message #2), and one company's out-of-memory reports (messages #37, #50). The objections concern the Execute scope and the SELECT-only limit.

### Precedents retrieved

- P1: [cursors with prepared statements](https://commitfest.postgresql.org/patch/1667/) (cf, Returned with feedback, 2018-12-01) - used in the review; reason summarised from its thread
- P2: [loop over unbound cursor](https://commitfest.postgresql.org/patch/2558/) (cf, Withdrawn, 2020-08-10) - judged not relevant; reason summarised from its thread
- P3: [PARALLEL flag for cursors](https://commitfest.postgresql.org/patch/1362/) (cf, Returned with feedback, 2018-02-07) - judged not relevant; reason summarised from its thread
- P4: [Add new protocol message to change GUCs to be able to change protocol extension parameters](https://commitfest.postgresql.org/patch/4736/) (cf, Withdrawn, 2025-02-22) - used in the review; reason summarised from its thread
- P5: [Use "protocol options" name instead of "protocol extensions" everywhere](https://commitfest.postgresql.org/patch/5341/) (cf, Rejected, 2025-03-11) - judged not relevant; reason summarised from its thread
- P6: [Improve cursor handling in pg_stat_statements](https://commitfest.postgresql.org/patch/3901/) (cf, Withdrawn, 2022-10-11) - judged not relevant; reason summarised from its thread
- P7: [Revert "Add GUC checks for ssl_min_protocol_version and ssl_max_protocol_version"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=414c2fd1e1c0ccd1af1cad046bef1264e3f4347f) (revert, Reverted, 2020-02-06) - judged not relevant
- P8: [temporary file usage report with extended protocol and unnamed portals](https://commitfest.postgresql.org/patch/5711/) (cf, Withdrawn, 2025-08-21) - judged not relevant; reason summarised from its thread
- P9: [Revert "Drop unnamed portal immediately after execution to completion"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=910690415b661186ae44e3b5e538e23eaa48de1b) (revert, Reverted, 2025-11-14) - used in the review
- P10: [Rearrange execution of PARAM_EXTERN Params for plpgsql's benefit.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=6719b238e8f0ea83c39d2b1728c7670cdaa34e06) (revert, Partially reverted, 2017-12-21) - judged not relevant
- P11: [Password identifiers, protocol aging and SCRAM protocol](https://commitfest.postgresql.org/patch/535/) (cf, Returned with feedback, 2016-03-22) - judged not relevant; reason summarised from its thread
- P12: [FETCH FIRST clause PERCENT option](https://commitfest.postgresql.org/patch/1759/) (cf, Returned with feedback, 2019-04-09) - judged not relevant
- P13: [Revert FETCH/MOVE int64 patch. Was using incorrect checks for fetch/move in scan.l.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=0e20c485617c3bcde8d9f0759d5ee1ef1c442004) (revert, Reverted, 2006-09-03) - judged not relevant
- P14: [Revert doc change so we mention moving past the last row of a cursor.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=bf4bf092e581df1cfa830fd6bc02e036b05dbb5d) (revert, Reverted, 2003-02-04) - judged not relevant
- P15: [pg_stat_statements and extended query protocol](https://commitfest.postgresql.org/patch/4146/) (cf, Withdrawn, 2023-01-30) - judged not relevant
- P16: [Support tls-exporter as channel binding for TLSv1.3](https://commitfest.postgresql.org/patch/3850/) (cf, Returned with feedback, 2022-11-30) - judged not relevant
- P17: [Fix index-only scan plans, take 2.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=9a3ddeb519e8d91b670fa923619b212a9670389c) (revert, Partially reverted, 2022-01-03) - judged not relevant
- P18: [Extended protocol COPY FROM STDIN fix](https://commitfest.postgresql.org/patch/3180/) (cf, Returned with feedback, 2021-12-01) - judged not relevant
- P19: [Channel binding support for SCRAM-SHA-256](https://commitfest.postgresql.org/patch/1153/) (cf, Returned with feedback, 2017-09-26) - judged not relevant
- P20: [Hooks and API to allow loadable wire protocol extensions](https://commitfest.postgresql.org/patch/3018/) (cf, Returned with feedback, 2021-10-01) - judged not relevant
- P21: [Dynamic result sets from procedures](https://commitfest.postgresql.org/patch/2911/) (cf, Withdrawn, 2023-10-30) - used in the review
- P22: [Per-table storage parameters for TableAM/IndexAM extensions](https://commitfest.postgresql.org/patch/3495/) (cf, Returned with feedback, 2022-08-02) - judged not relevant
- P23: [minor fix in CancelVirtualTransaction](https://commitfest.postgresql.org/patch/1903/) (cf, Returned with feedback, 2019-02-03) - judged not relevant
- P24: [Handling better supported channel binding types for SSL implementations](https://commitfest.postgresql.org/patch/1491/) (cf, Rejected, 2018-08-06) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 14; wall time: 203 s; cost reported by the CLI: $5.93
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 5 from cache, 3 summarised now, 0 failed; quotes: 3 verified, 0 dropped; relevance pass: done, 4 of 24 kept
- Tone check: 126 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T02:47:31.867Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
